# Go Gin — Production-Grade RESTful Backend — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Anyone going from "I can write a hello-world Gin route" to "I design, build, secure, test, and ship a RESTful backend that survives real traffic and a growing team" — without an internet connection. Every concept is explained in *prose first* — what it is, the logic/why, what it's used for and when, how to use it, its parameters/props, best practices, and security recommendations — and *then* shown in heavily-commented, runnable code. Read top-to-bottom the first time; afterward use the Table of Contents as a reference. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **`github.com/gin-gonic/gin` v1.10+** (current in 2026) on **Go 1.25 / 1.26**. Gin's core API (routing, binding, middleware, context) has been stable since v1.7, so almost everything here applies from v1.7 onward. Go-language features used (generics 1.18, `log/slog` 1.21, `min`/`max`/`clear` 1.21, range-over-int 1.22, per-iteration loop variables 1.22, range-over-func iterators 1.23) are flagged where relevant. Where behaviour is version-sensitive it is flagged with **⚡ Version note**. The author is on **Windows 11**, so cross-platform notes (path separators, `.exe`, shells) are called out. Confirm exact APIs at `pkg.go.dev/github.com/gin-gonic/gin` when online.
>
> **This guide's place in the library:** It is the single deep reference for building a *production* Gin backend. It folds in the previous file-upload material and cross-references the companion guides where they go deeper:
> - `GO_LANG_AND_PATTERNS_GUIDE.md` — the Go language itself and the design patterns (clean architecture, repository, DI, factory, adapter, functional options) we *apply* here.
> - `JWT_AUTH_ARGON2_GUIDE.md` — token and password-hashing internals.
> - `POSTGRESQL_GUIDE.md` / `RELATIONAL_DB_DESIGN_GUIDE.md` — SQL and schema design.
> - `REDIS_GUIDE.md`, `DOCKER_GUIDE.md`, `GO_FILESYSTEM_OS_CLI_GUIDE.md` — caching, deployment, and file/OS plumbing.

---

## Table of Contents

1. [What Gin Is & Why Use It](#1-what-gin-is--why-use-it) **[B]**
2. [Setup, Engine & First Server](#2-setup-engine--first-server) **[B]**
3. [Routing Deep Dive](#3-routing-deep-dive) **[B/I]**
4. [The `*gin.Context` In Depth](#4-the-gincontext-in-depth) **[B/I]**
5. [RESTful API Design](#5-restful-api-design) **[I]**
6. [Request Binding, Validation & DTOs](#6-request-binding-validation--dtos) **[I]**
7. [File Uploads (Major Focus)](#7-file-uploads-major-focus) **[I/A]**
8. [The Two Folder Structures](#8-the-two-folder-structures) **[A]**
9. [Architecture & Design Patterns for Scale](#9-architecture--design-patterns-for-scale) **[A]**
10. [Middleware In Depth](#10-middleware-in-depth) **[I/A]**
11. [Authentication & Authorization](#11-authentication--authorization) **[A]**
12. [Database Layer](#12-database-layer) **[A]**
13. [Configuration & Secrets](#13-configuration--secrets) **[I/A]**
14. [Observability](#14-observability) **[A]**
15. [Centralized Error Handling](#15-centralized-error-handling) **[A]**
16. [Performance & Scale](#16-performance--scale) **[A]**
17. [Security Hardening](#17-security-hardening) **[A]**
18. [Testing](#18-testing) **[I/A]**
19. [Deployment](#19-deployment) **[A]**
20. [Complete Worked Example: Users + Orders API](#20-complete-worked-example-users--orders-api) **[A]**
21. [Gotchas & Best Practices](#21-gotchas--best-practices) **[A]**
22. [Study Path & Build-to-Learn Projects](#22-study-path--build-to-learn-projects)

---

## 1. What Gin Is & Why Use It

### 1.1 What Gin is **[B]**

**Gin** is a high-performance HTTP web framework written in Go. It does *not* replace Go's standard `net/http` — it *wraps* it. Under the hood a `*gin.Engine` is an `http.Handler`; you can hand it to `http.Server` directly. What Gin adds on top of `net/http` is a small set of things that you would otherwise write by hand for every API:

- A **fast router** based on a *radix tree* (the same data structure `julienschmidt/httprouter` popularised), which supports path parameters (`:id`), wildcards (`*path`), and route groups.
- A **middleware chain** with an explicit `Use()` / `Next()` / `Abort()` model.
- **Request binding & validation** driven by struct tags (`binding:"required,email"`), powered by `go-playground/validator/v10`.
- **Response rendering** helpers (`c.JSON`, `c.XML`, `c.String`, `c.File`, …) that set the right `Content-Type` and status in one call.
- A rich **`*gin.Context`** that carries the request, the response writer, per-request key/value storage, and chain control.

The mental model of one request flowing through Gin:

```
Incoming HTTP request
  → http.Server (stdlib) → gin.Engine.ServeHTTP
    → Router matches method+path in the radix tree
      → Middleware chain (global → group → route), each may call c.Next() or c.Abort()
        → Your handler reads via *gin.Context, renders a response
      ← Middleware "after c.Next()" code unwinds (logging, timing)
  ← Response written to the client
```

### 1.2 Why Gin — the logic behind it **[B]**

Three forces make Gin the default choice for Go REST APIs in 2026:

1. **Performance with near-zero allocations.** The radix-tree router matches routes in O(length-of-path) without scanning every route, and Gin reuses `Context` objects from a `sync.Pool` so the hot path allocates almost nothing. For an API serving thousands of requests per second, GC pressure and per-request allocations matter — fewer allocations means lower latency tails (p99) and less CPU spent in garbage collection.
2. **It stays close to `net/http`.** Because a Gin handler still has access to `c.Request` (`*http.Request`) and `c.Writer` (an enhanced `http.ResponseWriter`), you never lose the stdlib. You can drop down to raw `net/http` any time, reuse stdlib middleware, and your knowledge transfers.
3. **Batteries for APIs, not ceremony.** Binding+validation, JSON rendering, route groups, and file uploads are the four things every JSON API needs, and Gin gives you all four with minimal boilerplate.

### 1.3 Gin vs raw `net/http` **[B]**

| Concern | `net/http` (stdlib) | Gin |
|---|---|---|
| Router | `http.ServeMux` (pattern-based; method+wildcard routing added in Go 1.22) | Radix tree: `:id`, `*path`, groups, params |
| Middleware | Manual handler-wrapping functions | `Use()` + `c.Next()` / `c.Abort()` chain |
| Request binding | Manual `json.NewDecoder().Decode()` + manual error handling | `ShouldBindJSON/Query/Uri/Header` + struct tags |
| Validation | None built-in | `binding:"required,email,min=3"` via validator/v10 |
| JSON response | `json.Marshal` + set header + `w.Write` | `c.JSON(200, obj)` |
| File uploads | Manual multipart parsing | `c.FormFile`, `c.SaveUploadedFile`, `c.MultipartForm` |
| Per-request storage | `context.WithValue` only | `c.Set` / `c.Get` (typed getters) |

> **⚡ Version note:** Go 1.22 gave `net/http.ServeMux` method-aware patterns (`mux.HandleFunc("GET /users/{id}", …)`) and path values (`r.PathValue("id")`). For *simple* services the stdlib router is now genuinely viable. Gin still wins on middleware ergonomics, binding/validation, grouping, and the `Context` API — which is exactly what a *production* service leans on.

### 1.4 When to choose Gin — and when not **[B]**

Choose Gin when: you are building a REST/JSON API; you want composable middleware (auth, CORS, logging, rate-limiting); you need real file-upload handling; your team knows Go and wants to stay near `net/http`.

Look elsewhere when: you need gRPC (use `google.golang.org/grpc`), GraphQL (`gqlgen`), or a full server-rendered MVC framework. Among HTTP frameworks, the alternatives worth knowing are **Echo** (very similar to Gin), **Chi** (thin, 100% `net/http`-compatible, great for stdlib purists), and **Fiber** (Express-like but built on `fasthttp`, so *not* `net/http`-compatible — you lose stdlib middleware). For most new Go APIs, Gin or Echo are the standard picks.

> **Best practice:** Treat Gin as the *transport/delivery* layer only (§9). Your business logic should not import `gin` at all, so you could swap the web framework without rewriting the core. This single discipline is what keeps a large codebase maintainable.

---

## 2. Setup, Engine & First Server

### 2.1 Prerequisites & module init **[B]**

```bash
go version                       # need Go 1.23+ for everything in this guide
mkdir my-api && cd my-api
go mod init github.com/you/my-api  # module path doubles as the import prefix
go get github.com/gin-gonic/gin    # pulls Gin + validator + json codec into go.mod/go.sum
```

`go.mod` now records the dependency; `go.sum` records checksums so builds are reproducible. Commit both. Run `go mod tidy` often to drop unused deps.

### 2.2 `gin.Engine`, `gin.Default()` vs `gin.New()` **[B]**

The central object is **`*gin.Engine`** — it *is* the router, the middleware registry, and the `http.Handler`. There are two ways to create one, and the difference is purely *which middleware comes pre-installed*:

- **`gin.Default()`** returns an engine with two middleware already attached: `gin.Logger()` (writes a line per request to stdout) and `gin.Recovery()` (catches panics in any handler and turns them into a 500 instead of crashing the process). It's the friendliest start for development.
- **`gin.New()`** returns a *bare* engine with **no** middleware. You add exactly what you want. Use this in production so you control logging (structured, not Gin's text format) — but you must remember to add `gin.Recovery()` yourself, because an unrecovered panic kills the whole server.

```go
package main

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

func main() {
	// ── Option A: gin.Default() — Logger + Recovery preinstalled (dev-friendly) ──
	// r := gin.Default()

	// ── Option B: gin.New() — bare; you choose middleware (production) ───────────
	r := gin.New()
	r.Use(gin.Recovery()) // ALWAYS add Recovery in production — never skip it.
	// r.Use(myStructuredLogger())   // your own slog/zap logger (see §14)

	r.GET("/ping", func(c *gin.Context) {
		// c.JSON sets Content-Type: application/json and serialises the map.
		c.JSON(http.StatusOK, gin.H{"message": "pong"})
	})

	// r.Run is a convenience wrapper over http.ListenAndServe. For production,
	// build your own http.Server so you can set timeouts + graceful shutdown (§16).
	if err := r.Run(":8080"); err != nil {
		panic(err)
	}
}
```

```bash
go run .
# [GIN-debug] GET /ping --> main.main.func1 (2 handlers)
# [GIN-debug] Listening and serving HTTP on :8080
curl http://localhost:8080/ping     # {"message":"pong"}
```

> **`gin.H`** is just `map[string]any` — a convenience alias for ad-hoc JSON. For real responses prefer typed structs (§5.6) so the shape is documented and refactor-safe.

### 2.3 Run modes — debug / release / test **[B]**

Gin has three modes that control logging verbosity and a few optimisations. Set the mode **before** creating the engine.

```go
gin.SetMode(gin.ReleaseMode) // production: no debug route dump, minor optimisations
gin.SetMode(gin.DebugMode)   // default: verbose route registration logs + warnings
gin.SetMode(gin.TestMode)    // tests: suppresses output
// Or via environment, no code change:  GIN_MODE=release ./my-api
```

> **Best practice:** Drive the mode from config (§13), defaulting to release in deployed environments. Leaving an API in debug mode in production leaks route information and prints noisy logs.

### 2.4 A production-grade server skeleton (preview) **[I]**

`r.Run()` is fine for learning, but it gives you *no timeouts* and *no graceful shutdown*. A real server wraps the engine in an `http.Server` you control. We build this fully in §16; here's the shape so you start with the right habit:

```go
srv := &http.Server{
	Addr:              ":8080",
	Handler:           r,                // the gin.Engine is an http.Handler
	ReadHeaderTimeout: 5 * time.Second,  // defend against Slowloris (§17)
	ReadTimeout:       15 * time.Second,
	WriteTimeout:      15 * time.Second,
	IdleTimeout:       60 * time.Second,
}
// go srv.ListenAndServe()  ... then graceful shutdown on SIGINT/SIGTERM (§16.1)
```

### 2.5 Live reload in development with Air **[B]**

Go is compiled, so by default every code change means stopping the process, re-running `go build`/`go run`, and restarting by hand. During active development that loop gets tedious fast. **[Air](https://github.com/air-verse/air)** is the de-facto live-reload tool for Go web servers: it watches your source tree, and on any change it automatically rebuilds the binary and restarts it. You edit a handler, save, and within a second the new code is serving — the same feel as `nodemon` in Node or the Flask/Django auto-reloader in Python. **Use Air for local development only; never in production** (production runs the compiled binary directly behind graceful shutdown — §16/§19).

> **⚡ Note on the module path:** Air moved to the `air-verse` org. The current install path is `github.com/air-verse/air`. The old `github.com/cosmtrek/air` path still appears in older tutorials but is deprecated — prefer the new one.

**Install** (Go 1.25/1.26):

```bash
# Recommended: install the binary into $GOPATH/bin (make sure that's on your PATH).
go install github.com/air-verse/air@latest

# Verify (Windows produces air.exe; it's still invoked as `air`):
air -v

# Alternatives that don't require touching PATH:
#   go run github.com/air-verse/air@latest          # run without installing
#   curl -sSfL https://goblin.run/github.com/air-verse/air | sh   # script install (unix)
```

**Configure** with an `.air.toml` at your module root. Generate a default and then trim it:

```bash
air init   # writes a default .air.toml you can customize
```

A practical `.air.toml` for a Gin project (heavily commented):

```toml
# .air.toml — local dev live-reload config for a Gin API.
root = "."            # watch from the module root
tmp_dir = "tmp"       # where Air puts the rebuilt binary (add tmp/ to .gitignore)

[build]
  # The command Air runs to (re)build. On Windows, output an .exe.
  cmd = "go build -o ./tmp/main.exe ./cmd/api"   # adjust path to your main package
  bin = "tmp/main.exe"                            # the binary Air then runs
  # full_bin lets you pass env/flags when starting the binary, e.g. GIN_MODE:
  # full_bin = "GIN_MODE=debug ./tmp/main.exe"

  include_ext = ["go", "tmpl", "tpl", "html"]     # rebuild when these change
  exclude_dir = ["assets", "tmp", "vendor", "testdata", "node_modules"]
  exclude_regex = ["_test.go"]                      # don't rebuild on test-file edits
  delay = 200                                       # ms debounce after a change before rebuilding
  stop_on_error = true                              # keep the old binary running if the build fails
  kill_delay = "500ms"                              # wait for the old process to release the port

[log]
  time = true            # timestamp Air's own log lines

[misc]
  clean_on_exit = true   # delete tmp/ when Air stops
```

> On macOS/Linux use `bin = "tmp/main"` and `cmd = "go build -o ./tmp/main ./cmd/api"` (no `.exe`). If your `main` package is at the repo root instead of `./cmd/api`, use `.` as the build target.

**Run it** — just type `air` from the project root:

```bash
air
# Air builds ./cmd/api -> tmp/main.exe, starts it, and watches.
# Edit any .go file, save, and Air rebuilds + restarts automatically.
# Ctrl+C stops Air (and, with clean_on_exit, removes tmp/).
```

**Best practices and gotchas:**

- **Add `tmp/` to `.gitignore`** — it holds throwaway build artifacts.
- **Graceful shutdown still matters.** Air sends an interrupt to restart your process; if your server has graceful-shutdown wiring (§16.1) it drains cleanly between reloads and avoids "address already in use" on the next start. If you still see port-in-use errors, raise `kill_delay` so the old process fully releases the port first.
- **It is a dev tool, not a supervisor.** Don't ship Air in your Docker runtime stage or systemd unit — run the plain compiled binary there. A common setup uses Air only in a `docker-compose` *dev* override with the source bind-mounted.
- **Editor + Air together:** keep `gofmt`/`goimports`-on-save in your editor; Air handles rebuild/restart, your editor handles formatting.

---

## 3. Routing Deep Dive

### 3.1 Why the router is fast — the radix tree **[B/I]**

A naive router compares the incoming path against every registered route in turn — O(N) in the number of routes. Gin instead stores routes in a **radix tree** (a compressed prefix tree) *per HTTP method*. Shared prefixes (`/api/v1/users`, `/api/v1/orders`) are stored once; matching walks the tree by path segment, so lookup cost depends on the *length of the path*, not the *number of routes*. This is why a Gin app with 500 routes matches as fast as one with 5, and why route matching shows up as ~0 allocations in benchmarks. You don't manage the tree — you just register routes and Gin builds it.

### 3.2 HTTP method handlers **[B]**

```go
r := gin.New()
r.GET("/users", listUsers)          // read collection
r.POST("/users", createUser)        // create
r.GET("/users/:id", getUser)        // read one
r.PUT("/users/:id", replaceUser)    // full replace
r.PATCH("/users/:id", patchUser)    // partial update
r.DELETE("/users/:id", deleteUser)  // remove
r.HEAD("/users", headUsers)         // headers only
r.OPTIONS("/users", optionsUsers)   // capabilities / CORS preflight

r.Any("/webhook", handleWebhook)    // matches every method
r.Match([]string{"GET", "POST"}, "/flex", flexHandler) // a chosen subset
```

### 3.3 Route parameters and wildcards **[B]**

A **named parameter** `:name` matches exactly one path segment (no slashes). A **wildcard** `*name` matches the rest of the path *including* slashes, and must be the last segment.

```go
// Named param — one segment. /users/42 → "42"
r.GET("/users/:id", func(c *gin.Context) {
	id := c.Param("id")
	c.JSON(200, gin.H{"id": id})
})

// Multiple named params:
r.GET("/users/:id/orders/:orderID", func(c *gin.Context) {
	c.JSON(200, gin.H{"user": c.Param("id"), "order": c.Param("orderID")})
})

// Wildcard — captures everything after the prefix, slashes included.
// /files/images/cat.jpg → "/images/cat.jpg"
r.GET("/files/*filepath", func(c *gin.Context) {
	c.String(200, "file: %s", c.Param("filepath"))
})
```

> **Gotcha:** You cannot register conflicting routes that the radix tree can't disambiguate, e.g. `/users/:id` and `/users/new` will *panic at startup* in older Gin if they collide on a node — register the static route style carefully, or use a query/sub-path. Gin v1.10's router handles many of these, but keep static and param routes from overlapping ambiguously.

### 3.4 Query parameters **[B]**

Query parameters are read from the URL's `?key=value` part. They never panic on missing keys — you get `""` or a provided default.

```go
// /search?q=gin&page=2&limit=10
r.GET("/search", func(c *gin.Context) {
	q := c.Query("q")                       // "" if absent
	page := c.DefaultQuery("page", "1")     // "2" or default "1"
	limit, has := c.GetQuery("limit")       // value + bool "was it present?"

	// Repeated keys: /tags?t=a&t=b → []string{"a","b"}
	tags := c.QueryArray("t")
	// Map syntax: /f?ids[a]=1&ids[b]=2 → map[string]string{"a":"1","b":"2"}
	ids := c.QueryMap("ids")

	c.JSON(200, gin.H{"q": q, "page": page, "limit": limit, "has": has, "tags": tags, "ids": ids})
})
```

For structured pagination/filtering, bind the whole query into a struct instead (§6.3).

### 3.5 Route groups & versioning **[B/I]**

A **route group** shares a URL prefix and (optionally) middleware across many routes. Groups are how you version an API (`/api/v1`), apply auth to a subtree, and keep registration tidy. Groups nest.

```go
r := gin.New()

v1 := r.Group("/api/v1")
{ // braces are cosmetic — they visually scope the group's routes
	v1.GET("/products", listProducts)
	v1.POST("/products", createProduct)

	// Nested group with its own middleware — only admin routes get AdminOnly().
	admin := v1.Group("/admin")
	admin.Use(JWTAuth(), AdminOnly()) // middleware applies to everything below
	{
		admin.DELETE("/users/:id", deleteUser)
	}
}

// A separate version can evolve independently:
v2 := r.Group("/api/v2")
v2.GET("/products", listProductsV2)
```

### 3.6 Multiple handlers, 404/405, trailing slashes **[I]**

Every route accepts a *chain* of handlers — middleware then the final handler. `NoRoute`/`NoMethod` customise 404/405 responses.

```go
// Per-route middleware: auth runs, then the handler.
r.POST("/admin/reset", JWTAuth(), AdminOnly(), resetHandler)

// Custom 404 / 405 with a consistent JSON envelope:
r.NoRoute(func(c *gin.Context) {
	c.JSON(404, gin.H{"error": gin.H{"code": "not_found", "message": "route not found"}})
})
r.NoMethod(func(c *gin.Context) {
	c.JSON(405, gin.H{"error": gin.H{"code": "method_not_allowed"}})
})

// Trailing-slash behaviour: by default Gin redirects /users/ → /users (301).
// In APIs you often want this OFF to avoid surprising redirects:
r.RedirectTrailingSlash = false
r.HandleMethodNotAllowed = true // so NoMethod (405) fires instead of 404
```

| Router knob | Use |
|---|---|
| `r.Group(prefix, mw...)` | Prefix + shared middleware |
| `r.Use(mw...)` | Global middleware (all routes) |
| `r.NoRoute(h)` | Custom 404 |
| `r.NoMethod(h)` | Custom 405 (needs `HandleMethodNotAllowed = true`) |
| `r.RedirectTrailingSlash` | Auto-redirect `/path/` ↔ `/path` |
| `r.MaxMultipartMemory` | RAM buffer for uploads (§7) |

---

## 4. The `*gin.Context` In Depth

`*gin.Context` is *the* object every handler and middleware receives. It is the request, the response, the per-request scratchpad, and the chain controller rolled into one. Understanding it well is most of understanding Gin.

### 4.1 The context lifecycle & the sync.Pool **[I]**

For each incoming request Gin takes a `Context` from an internal `sync.Pool`, resets it, runs the middleware/handler chain, then returns it to the pool. This pooling is *why Gin allocates so little* — but it has a critical consequence:

> **Gotcha (the most important Context rule):** The `*gin.Context` is **only valid for the duration of the request**. Never store it, and never use it inside a goroutine that outlives the handler. If you must do background work, either finish before returning, or **copy** the context with `cCp := c.Copy()` (a read-only snapshot safe to pass to a goroutine) and pass `c.Request.Context()` for cancellation.

```go
func handler(c *gin.Context) {
	cCp := c.Copy() // safe snapshot for the goroutine
	go func() {
		// Use cCp (read-only) here — NOT c. Using c risks reading a recycled context.
		time.Sleep(2 * time.Second)
		log.Println("path was", cCp.Request.URL.Path)
	}()
	c.JSON(202, gin.H{"status": "accepted"})
}
```

### 4.2 Reading request data **[B]**

```go
func read(c *gin.Context) {
	id := c.Param("id")               // route param  /users/:id
	q := c.Query("q")                 // query string ?q=...
	auth := c.GetHeader("Authorization") // request header
	ct := c.ContentType()             // parsed Content-Type
	ip := c.ClientIP()                // honours trusted proxies (§4.6)
	method := c.Request.Method        // raw stdlib request is always available
	host := c.Request.Host
	ctx := c.Request.Context()        // the Go context for DB calls/cancellation (§12)
	_ = id; _ = q; _ = auth; _ = ct; _ = ip; _ = method; _ = host; _ = ctx
}
```

> **Gotcha:** The request body is a *stream* — it can be read only once. After `ShouldBindJSON` consumes it, a second read returns nothing. If middleware *and* a handler both need the body, use `c.ShouldBindBodyWith` / `c.ShouldBindBodyWithJSON`, which buffer the body in the context so it can be re-read.

### 4.3 Rendering responses **[B]**

Each render helper sets the appropriate `Content-Type` and writes status + body.

```go
c.JSON(200, gin.H{"ok": true})        // application/json
c.JSON(200, myStruct)                 // structs serialise via json tags
c.PureJSON(200, obj)                  // like JSON but does NOT escape <, >, & (use carefully)
c.IndentedJSON(200, obj)              // pretty JSON — slower; debug only
c.String(200, "hello %s", name)       // text/plain
c.XML(200, obj)                       // application/xml
c.YAML(200, obj)                      // application/yaml
c.Data(200, "image/png", pngBytes)   // raw bytes with explicit content type
c.File("/path/report.pdf")            // serve a file from disk
c.Redirect(302, "/new")               // Location header + status
c.Status(204)                         // status only, no body
```

> **Gotcha:** A response can be written only once. Calling `c.JSON` twice — or calling `c.JSON` after `c.AbortWithStatusJSON` — produces a `superfluous WriteHeader call` warning and a malformed response. After you render, **`return`**.

### 4.4 Chain control: `Next`, `Abort`, and the "after" phase **[I]**

Middleware uses `c.Next()` to pass control *down* the chain and `c.Abort()` to stop it. The subtle, powerful part: code **after** `c.Next()` runs on the way back *out*, after the handler has produced its response. That's how timing, logging, and cleanup middleware work.

```go
func timing() gin.HandlerFunc {
	return func(c *gin.Context) {
		start := time.Now()
		c.Next()                         // ← run the rest of the chain (incl. the handler)
		dur := time.Since(start)         // ← runs AFTER the handler returns
		status := c.Writer.Status()      // the status the handler set
		log.Printf("%s %s %d %v", c.Request.Method, c.Request.URL.Path, status, dur)
	}
}
```

- `c.Abort()` sets a flag so no *later* handlers run, but it does **not** stop the current function — you must `return` after it.
- `c.AbortWithStatusJSON(code, body)` aborts *and* writes a JSON error in one call (the usual move in auth middleware).
- `c.IsAborted()` reports whether the chain was aborted.

### 4.5 Per-request storage: `Set` / `Get` **[B]**

Middleware stores values on the context; downstream handlers read them. This is how the auth middleware passes the authenticated user ID to handlers without globals.

```go
// In middleware:
c.Set("userID", 42)
c.Set("role", "admin")

// In a handler — generic Get returns (any, bool):
v, ok := c.Get("userID")
if !ok {
	c.AbortWithStatusJSON(401, gin.H{"error": "unauthenticated"})
	return
}
userID := v.(int)

// Typed convenience getters (return the zero value if absent/wrong type):
role := c.GetString("role")
n := c.GetInt("userID")
_ = role; _ = n
```

> **Best practice:** Define **typed constant keys** for context values rather than bare strings sprinkled around (`const ctxUserID = "userID"`), and wrap `Get` in small typed accessors so the rest of the code is refactor-safe and can't fat-finger a key.

### 4.6 Client IP & trusted proxies (security) **[I]**

`c.ClientIP()` returns the caller's IP, but behind a load balancer the *direct* peer is the proxy, and the real client appears in `X-Forwarded-For` / `X-Real-IP`. An attacker can spoof those headers unless you tell Gin which proxies to trust. Misconfigured, this breaks rate-limiting and audit logs.

```go
r := gin.New()
// Only trust forwarded headers from these proxy networks (your LB/ingress):
_ = r.SetTrustedProxies([]string{"10.0.0.0/8", "172.16.0.0/12"})
// If you have NO proxy (direct exposure), trust none so headers can't spoof IPs:
// _ = r.SetTrustedProxies(nil)
```

> **Security recommendation:** Never leave the default "trust all proxies" in production. Set the exact CIDR of your ingress, or `nil` if directly exposed. `ClientIP()` is only as trustworthy as this setting.

### 4.7 Status code quick reference **[B]**

| Constant | Code | Meaning / when |
|---|---|---|
| `StatusOK` | 200 | Successful GET/PUT/PATCH with a body |
| `StatusCreated` | 201 | POST created a resource (return it + `Location`) |
| `StatusAccepted` | 202 | Async work queued |
| `StatusNoContent` | 204 | Success, no body (DELETE) |
| `StatusBadRequest` | 400 | Malformed request / unparseable |
| `StatusUnauthorized` | 401 | Not authenticated |
| `StatusForbidden` | 403 | Authenticated but not allowed |
| `StatusNotFound` | 404 | Resource doesn't exist |
| `StatusConflict` | 409 | Duplicate / version conflict |
| `StatusUnprocessableEntity` | 422 | Well-formed but failed validation |
| `StatusTooManyRequests` | 429 | Rate limit exceeded (`Retry-After`) |
| `StatusInternalServerError` | 500 | Unexpected server fault |
| `StatusServiceUnavailable` | 503 | Down for maintenance / overloaded |

---

## 5. RESTful API Design

A framework gives you the *mechanics*; REST gives you the *conventions* that make an API predictable to consumers. Getting the design right early saves painful breaking changes later.

### 5.1 REST principles & resource modeling **[I]**

**REST** (Representational State Transfer) models your system as **resources** (nouns) that clients manipulate through a uniform set of HTTP **methods** (verbs). The core ideas you actually apply:

- **Resources are nouns, not actions.** `/users`, `/orders`, `/orders/42/items` — never `/getUser` or `/createOrder`. The *method* is the verb.
- **Statelessness.** Each request carries everything needed to process it (auth token, parameters); the server keeps no per-client session in memory. This is what lets you run many identical instances behind a load balancer (§16) — any instance can serve any request.
- **Uniform interface.** The same method means the same thing everywhere: GET reads, POST creates, PUT replaces, PATCH partially updates, DELETE removes.
- **Representations.** A resource is transferred as a representation (usually JSON). The same resource could be offered as JSON or XML via content negotiation (§5.7).

Model **collections** and **members**: `/orders` is a collection; `/orders/42` is a member; `/orders/42/items` is a sub-collection. Keep nesting shallow (one level is usually enough) — deep nesting (`/users/1/orders/2/items/3/reviews/4`) becomes unwieldy; prefer top-level resources with filters (`/items?order_id=2`).

### 5.2 HTTP methods, status codes & idempotency **[I]**

| Method | Purpose | Safe? | Idempotent? | Typical success |
|---|---|---|---|---|
| GET | Read | yes | yes | 200 |
| POST | Create / non-idempotent action | no | **no** | 201 (+`Location`) |
| PUT | Replace whole resource | no | **yes** | 200 / 204 |
| PATCH | Partial update | no | usually | 200 |
| DELETE | Remove | no | **yes** | 204 |

- **Safe** = no side effects (GET/HEAD). **Idempotent** = doing it N times equals doing it once. PUT/DELETE are idempotent (deleting an already-deleted resource still ends in "gone"); POST is not (two POSTs create two resources).
- **Idempotency keys:** Because POST isn't idempotent, clients that retry on a flaky network can double-create. The pattern: the client sends an `Idempotency-Key: <uuid>` header; the server records the key with the result and returns the *same* result for a repeated key instead of re-processing. Essential for payments and order creation.

```go
// Idempotency middleware sketch (store keyed results in Redis with a TTL):
func Idempotency(store IdempotencyStore) gin.HandlerFunc {
	return func(c *gin.Context) {
		if c.Request.Method != http.MethodPost {
			c.Next()
			return
		}
		key := c.GetHeader("Idempotency-Key")
		if key == "" {
			c.Next()
			return
		}
		if cached, ok := store.Get(c.Request.Context(), key); ok {
			c.Data(cached.Status, "application/json", cached.Body) // replay prior result
			c.Abort()
			return
		}
		c.Next() // (a response recorder + store.Save on the way out completes the pattern)
	}
}
```

### 5.3 URL design & versioning **[I]**

- Lowercase, hyphenated, plural nouns: `/api/v1/order-items`.
- **Version in the path** (`/api/v1/...`). It's the most explicit, cache-friendly, and easiest to route via groups. Header-based versioning (`Accept: application/vnd.app.v2+json`) is "purer" REST but harder to test and debug. Most teams choose path versioning — keep `/v1` stable and add `/v2` for breaking changes.
- Never break a published version. Additive changes (new optional field, new endpoint) are safe within a version; removing/renaming fields or changing types is a breaking change → new version.

### 5.4 Pagination, filtering, sorting **[I]**

Never return an unbounded collection — it will eventually OOM your server or time out. Always paginate. Two styles:

- **Offset/limit** (`?page=2&limit=20`): simple, supports "jump to page N", but slow for deep pages on large tables (`OFFSET 100000` scans and discards rows) and can skip/duplicate rows if data changes between pages.
- **Cursor/keyset** (`?limit=20&after=<opaque-cursor>`): the cursor encodes the last seen sort key; the next query is `WHERE id > cursor ORDER BY id LIMIT 20`. Scales to huge tables and is stable under inserts. Preferred for large/feed-like data; the trade-off is no random page access.

```go
type ListQuery struct {
	Page  int    `form:"page"  binding:"omitempty,min=1"`
	Limit int    `form:"limit" binding:"omitempty,min=1,max=100"` // CAP the limit — never trust the client
	Sort  string `form:"sort"  binding:"omitempty,oneof=created_at -created_at name -name"` // allowlist!
	Q     string `form:"q"     binding:"omitempty,max=100"`
}
```

> **Security recommendation:** Always *allowlist* sort fields and directions with `oneof` (above) and map them to real column names in code. Never interpolate a client-supplied sort string into SQL — that is an injection and an information-disclosure vector.

A consistent paginated envelope:

```go
type Page[T any] struct { // Go generics (1.18+) make this reusable across resources
	Data       []T   `json:"data"`
	Page       int   `json:"page"`
	Limit      int   `json:"limit"`
	TotalItems int64 `json:"total_items"`
	TotalPages int   `json:"total_pages"`
}
```

### 5.5 Consistent JSON envelopes for success & error **[I]**

Pick **one** success shape and **one** error shape and use them everywhere. Consumers (and your own frontend) then write parsing code once. A widely used convention:

```go
// Success: return the resource directly (200/201) or a Page[T] for collections.
// Error: a single, predictable envelope.
type ErrorBody struct {
	Code    string       `json:"code"`              // machine-readable: "not_found", "validation_failed"
	Message string       `json:"message"`           // human-readable summary
	Details []FieldError `json:"details,omitempty"` // per-field validation problems
	TraceID string       `json:"trace_id,omitempty"`// correlation ID for support (§14)
}
type ErrorResponse struct {
	Error ErrorBody `json:"error"`
}
```

```json
{ "error": { "code": "validation_failed", "message": "request body is invalid",
  "details": [{ "field": "email", "message": "must be a valid email address" }],
  "trace_id": "9f1c..." } }
```

### 5.6 DTOs vs domain models (why separate them) **[I]**

A **DTO** (Data Transfer Object) is the shape on the *wire* — the JSON the client sends/receives. A **domain model** is the shape in your *business logic*. Beginners use one struct for both; that couples your API contract to your internals and creates real problems:

- **Security:** a single struct risks *over-posting* (a client sets `IsAdmin:true` or `Balance:9999` on a field they shouldn't control) and *over-exposure* (you accidentally serialise a `PasswordHash` field). Separate request/response DTOs let you expose exactly the fields each direction needs.
- **Stability:** you can refactor the domain model (rename a column, split a struct) without breaking the API, because the DTO mapping absorbs the change.
- **Validation lives on the request DTO**, business invariants live on the domain model — clean separation.

```go
// Request DTO — only what the client may set. No ID, no timestamps, no role.
type CreateUserRequest struct {
	Email    string `json:"email"    binding:"required,email"`
	Name     string `json:"name"     binding:"required,min=2,max=100"`
	Password string `json:"password" binding:"required,min=10"`
}

// Domain model — internal; contains things the client must never set or see.
type User struct {
	ID           string
	Email        string
	Name         string
	PasswordHash string // NEVER serialised to a response
	Role         string
	CreatedAt    time.Time
}

// Response DTO — exactly what we return. PasswordHash is structurally impossible to leak.
type UserResponse struct {
	ID        string    `json:"id"`
	Email     string    `json:"email"`
	Name      string    `json:"name"`
	Role      string    `json:"role"`
	CreatedAt time.Time `json:"created_at"`
}

// Mapper (domain → response DTO). Keep these tiny and explicit.
func toUserResponse(u User) UserResponse {
	return UserResponse{ID: u.ID, Email: u.Email, Name: u.Name, Role: u.Role, CreatedAt: u.CreatedAt}
}
```

### 5.7 Content negotiation & HATEOAS (note) **[I]**

- **Content negotiation:** clients may send `Accept: application/json` or `application/xml`; Gin's `c.Negotiate` can serve the format the client asked for. In practice most JSON APIs just always return JSON and reject other `Accept` values — full negotiation is rarely worth the complexity.

```go
c.Negotiate(http.StatusOK, gin.Negotiate{
	Offered: []string{gin.MIMEJSON, gin.MIMEXML},
	Data:    payload, // rendered as JSON or XML depending on the Accept header
})
```

- **HATEOAS** (Hypermedia As The Engine Of Application State) is the REST ideal where responses embed links to related actions (`"_links": {"cancel": "/orders/42/cancel"}`). It's elegant but rarely fully adopted; most production APIs are "REST-ish" (resources + methods + status codes) without hypermedia. Know the term; don't feel obligated to implement it.

---

## 6. Request Binding, Validation & DTOs

Binding reads the request (JSON body, query string, URI params, headers, multipart form) into a Go struct and validates it via struct tags — replacing dozens of lines of manual decoding and checking. Gin uses `go-playground/validator/v10` for the validation tags.

### 6.1 `Bind*` vs `ShouldBind*` — always prefer ShouldBind **[I]**

There are two families. The `MustBind`/`Bind*` family (`c.BindJSON`, `c.Bind`) **automatically** writes a `400` and aborts on failure — you lose control of the error format and status. The `ShouldBind*` family returns the error and leaves the response to you, so you can produce your consistent envelope (§5.5) and choose `422` vs `400`. **Always use `ShouldBind*` in production.**

```go
func createUser(c *gin.Context) {
	var req CreateUserRequest
	if err := c.ShouldBindJSON(&req); err != nil {
		// We control status + shape — return a 422 with field-level details (§6.6).
		respondValidationError(c, err)
		return
	}
	// req is now validated; safe to use.
	c.JSON(http.StatusCreated, gin.H{"name": req.Name})
}
```

### 6.2 Binding by source **[I]**

Each source has a dedicated method and uses a specific struct tag:

```go
// JSON body  → `json` tag
type Body struct{ Name string `json:"name" binding:"required"` }
c.ShouldBindJSON(&body)

// Query string → `form` tag
type Query struct{ Page int `form:"page" binding:"omitempty,min=1"` }
c.ShouldBindQuery(&query)

// URI path params → `uri` tag
type URI struct{ ID string `uri:"id" binding:"required,uuid"` }
c.ShouldBindUri(&uri) // for route /users/:id

// Headers → `header` tag
type Headers struct{ APIVersion string `header:"X-Api-Version" binding:"required"` }
c.ShouldBindHeader(&headers)

// Auto by Content-Type (JSON, form, XML, …) → use `form` for multipart/urlencoded
c.ShouldBind(&anything)
```

> **Gotcha:** `ShouldBind` (no suffix) picks the binder from `Content-Type` and from the HTTP method. For a JSON body, call `ShouldBindJSON` *explicitly* so you don't accidentally hit the form binder. Query binding uses the `form` tag, **not** `json`.

### 6.3 The validator tag reference **[I]**

| Tag | Meaning |
|---|---|
| `required` | Must be present and non-zero (see pointer gotcha §6.5) |
| `omitempty` | Skip remaining validations if the value is the zero value |
| `min=N` / `max=N` | String length or numeric bound |
| `len=N` | Exact length |
| `gt`/`gte`/`lt`/`lte=N` | Numeric comparisons |
| `eqfield=Other` / `nefield` | Cross-field equal / not-equal (e.g. password confirm) |
| `email`, `url`, `uri`, `ip`, `uuid`, `datetime=2006-01-02` | Format checks |
| `oneof=a b c` | Enum allowlist |
| `alpha`, `alphanum`, `numeric` | Character-class checks |
| `dive` | Descend into each slice/map element to validate it |
| `required_if`, `required_with`, `excluded_with` | Conditional requirements |

```go
type RegisterRequest struct {
	Email    string   `json:"email"     binding:"required,email"`
	Password string   `json:"password"  binding:"required,min=10,max=128"`
	Confirm  string   `json:"confirm"   binding:"required,eqfield=Password"`
	Age      int      `json:"age"       binding:"required,gte=13,lte=120"`
	Role     string   `json:"role"      binding:"omitempty,oneof=user editor admin"`
	Tags     []string `json:"tags"      binding:"omitempty,max=10,dive,min=1,max=20"`
}
```

### 6.4 Custom validators **[I]**

Register custom rules once at startup against the underlying validator engine. Use them for domain rules the built-ins don't cover (slugs, no-whitespace, business-specific formats).

```go
import (
	"github.com/gin-gonic/gin/binding"
	"github.com/go-playground/validator/v10"
)

func registerValidators() {
	v, ok := binding.Validator.Engine().(*validator.Validate)
	if !ok {
		return
	}
	// "slug": lowercase letters, digits, hyphens; non-empty.
	_ = v.RegisterValidation("slug", func(fl validator.FieldLevel) bool {
		s := fl.Field().String()
		if s == "" {
			return false
		}
		for _, r := range s {
			if !((r >= 'a' && r <= 'z') || (r >= '0' && r <= '9') || r == '-') {
				return false
			}
		}
		return true
	})
}
// Usage: Slug string `json:"slug" binding:"required,slug,max=80"`
```

### 6.5 The `required` + zero-value gotcha **[I]**

`required` treats the **zero value** as "missing". For an `int` field, `0` is the zero value, so `{"stock": 0}` *fails* `required` even though the client explicitly sent 0. Same for `false` on a `bool`. When zero/false is a legitimate value you must distinguish "absent" from "present-and-zero" — use a **pointer** with `omitempty`:

```go
type UpdateProduct struct {
	// Pointer: nil = "field absent", non-nil = "client sent this value (even 0)".
	Stock *int     `json:"stock"  binding:"omitempty,gte=0"`
	Price *float64 `json:"price"  binding:"omitempty,gt=0"`
	Name  *string  `json:"name"   binding:"omitempty,min=2"`
}
// In the service, only apply non-nil fields — this is the natural PATCH semantics.
```

### 6.6 Formatting validation errors into the envelope **[I]**

Raw validator errors (`Key: 'X.Email' Error:Field validation for 'Email' failed on the 'email' tag`) are unfriendly. Translate them into per-field messages matching the §5.5 envelope.

```go
type FieldError struct {
	Field   string `json:"field"`
	Message string `json:"message"`
}

func toFieldErrors(err error) []FieldError {
	var ve validator.ValidationErrors
	if !errors.As(err, &ve) {
		// Not a validation error (e.g. malformed JSON syntax) → single generic entry.
		return []FieldError{{Field: "body", Message: err.Error()}}
	}
	out := make([]FieldError, 0, len(ve))
	for _, fe := range ve {
		out = append(out, FieldError{Field: lowerFirst(fe.Field()), Message: msgFor(fe)})
	}
	return out
}

func msgFor(fe validator.FieldError) string {
	switch fe.Tag() {
	case "required":
		return "this field is required"
	case "email":
		return "must be a valid email address"
	case "min":
		return fmt.Sprintf("must be at least %s", fe.Param())
	case "max":
		return fmt.Sprintf("must be at most %s", fe.Param())
	case "oneof":
		return fmt.Sprintf("must be one of: %s", fe.Param())
	default:
		return fmt.Sprintf("failed the %q rule", fe.Tag())
	}
}

func respondValidationError(c *gin.Context, err error) {
	c.JSON(http.StatusUnprocessableEntity, ErrorResponse{Error: ErrorBody{
		Code:    "validation_failed",
		Message: "request body is invalid",
		Details: toFieldErrors(err),
	}})
}
```

> **Security recommendation:** Don't echo raw decoder errors that may reveal internal types in production. Map known tags to safe messages (above); for the unknown/malformed-JSON case, return a generic "request body is invalid" rather than the parser's internal text.

---

## 7. File Uploads (Major Focus)

File uploads use `multipart/form-data`. Gin parses the multipart body lazily; `r.MaxMultipartMemory` controls how many bytes are buffered in RAM before the rest spills to a temp file on disk. Uploads are a top security-risk surface, so this section pairs each capability with its hardening.

### 7.1 The multipart memory limit **[I]**

```go
r := gin.New()
r.MaxMultipartMemory = 8 << 20 // 8 MiB buffered in RAM; the remainder goes to a temp file
```

This is *not* a hard cap on upload size — large files still land on disk. Enforce a **hard total request size** separately so a client can't fill your disk (§7.6 / §17).

### 7.2 Single file upload **[I]**

```go
func uploadSingle(c *gin.Context) {
	fh, err := c.FormFile("file") // "file" = the multipart field name the client uses
	if err != nil {
		c.JSON(400, gin.H{"error": "file field is required"})
		return
	}
	// SaveUploadedFile writes the file to dst. The directory must already exist.
	// WARNING: fh.Filename is attacker-controlled — never use it raw (§7.4).
	dst := filepath.Join("uploads", "raw", filepath.Base(fh.Filename))
	if err := c.SaveUploadedFile(fh, dst); err != nil {
		c.JSON(500, gin.H{"error": "failed to save file"})
		return
	}
	c.JSON(201, gin.H{"filename": fh.Filename, "size": fh.Size})
}
```

### 7.3 Multiple files & mixed form fields **[I]**

```go
func uploadMultiple(c *gin.Context) {
	form, err := c.MultipartForm() // parses the whole multipart form
	if err != nil {
		c.JSON(400, gin.H{"error": "invalid multipart form"})
		return
	}
	files := form.File["files"] // []*multipart.FileHeader
	if len(files) == 0 {
		c.JSON(400, gin.H{"error": "at least one file required"})
		return
	}
	if len(files) > 10 { // bound the count — defends against zip-bomb-style request floods
		c.JSON(400, gin.H{"error": "maximum 10 files per request"})
		return
	}
	for _, fh := range files {
		_ = fh // validate + save each (see §7.4)
	}
	c.JSON(201, gin.H{"count": len(files)})
}

// Mixing text fields with a file: bind text via the form tag, fetch the file separately.
type UploadProductForm struct {
	Name  string  `form:"name"  binding:"required,min=2"`
	Price float64 `form:"price" binding:"required,gt=0"`
}

func uploadProduct(c *gin.Context) {
	var f UploadProductForm
	if err := c.ShouldBind(&f); err != nil { // reads multipart text fields
		c.JSON(422, gin.H{"error": err.Error()})
		return
	}
	fh, err := c.FormFile("image")
	if err != nil {
		c.JSON(400, gin.H{"error": "image is required"})
		return
	}
	_ = fh // validate + save
	c.JSON(201, gin.H{"name": f.Name, "price": f.Price})
}
```

```bash
curl -X POST http://localhost:8080/api/v1/products/upload \
  -F "name=Widget Pro" -F "price=29.99" \
  -F "image=@/path/photo.jpg;type=image/jpeg"
```

### 7.4 Validating uploads (security-critical) **[A]**

**Never trust the client's filename, extension, or `Content-Type` header.** All three are attacker-controlled. The defence is: cap the size, detect the real type by reading magic bytes, generate your own safe filename, and store outside the webroot.

```go
package upload

import (
	"errors"
	"fmt"
	"io"
	"mime"
	"mime/multipart"
	"net/http"
	"os"
	"path/filepath"

	"github.com/google/uuid"
)

// Allowlist: detected MIME type → the extension WE assign (never the client's).
var allowedImage = map[string]string{
	"image/jpeg": ".jpg",
	"image/png":  ".png",
	"image/webp": ".webp",
	"image/gif":  ".gif",
}

const maxImageSize = 5 << 20 // 5 MiB

var (
	ErrTooLarge  = errors.New("file too large")
	ErrEmpty     = errors.New("file is empty")
	ErrBadType   = errors.New("file type not allowed")
)

// ValidateAndSaveImage checks size + true MIME and writes the file under a safe,
// unique name. Returns the stored filename (NOT a path the client controls).
func ValidateAndSaveImage(fh *multipart.FileHeader, dir string) (string, error) {
	// 1. Size — cheap rejection first.
	if fh.Size == 0 {
		return "", ErrEmpty
	}
	if fh.Size > maxImageSize {
		return "", fmt.Errorf("%w: max %d bytes", ErrTooLarge, maxImageSize)
	}

	// 2. Open the uploaded file.
	src, err := fh.Open()
	if err != nil {
		return "", fmt.Errorf("open upload: %w", err)
	}
	defer src.Close()

	// 3. Detect the REAL type from the first 512 bytes (magic numbers).
	//    Do NOT use fh.Header.Get("Content-Type") — the client sets it.
	head := make([]byte, 512)
	n, err := src.Read(head)
	if err != nil && !errors.Is(err, io.EOF) {
		return "", fmt.Errorf("read head: %w", err)
	}
	detected := http.DetectContentType(head[:n])
	mediaType, _, _ := mime.ParseMediaType(detected) // strip "; charset=..."
	ext, ok := allowedImage[mediaType]
	if !ok {
		return "", fmt.Errorf("%w: %s", ErrBadType, mediaType)
	}

	// 4. Generate OUR filename — defeats path traversal (e.g. "../../etc/passwd")
	//    and collisions. Extension comes from our allowlist, not the upload.
	name := uuid.NewString() + ext

	// 5. Ensure the directory exists, then stream the full file to disk.
	if err := os.MkdirAll(dir, 0o750); err != nil {
		return "", fmt.Errorf("mkdir: %w", err)
	}
	if _, err := src.Seek(0, io.SeekStart); err != nil { // rewind past the 512 we read
		return "", fmt.Errorf("seek: %w", err)
	}
	dst, err := os.Create(filepath.Join(dir, name))
	if err != nil {
		return "", fmt.Errorf("create: %w", err)
	}
	defer dst.Close()
	// io.CopyN bounds the write so a lying fh.Size can't blow past the cap.
	if _, err := io.CopyN(dst, src, maxImageSize+1); err != nil && !errors.Is(err, io.EOF) {
		_ = os.Remove(dst.Name()) // clean up a partial write
		return "", fmt.Errorf("write: %w", err)
	}
	return name, nil
}
```

### 7.5 Verifying it's a *real*, sane image **[A]**

`http.DetectContentType` only checks the header bytes — an attacker can prepend a valid JPEG magic number to a malicious payload. For stricter assurance, decode the image and reject suspicious dimensions ("decompression bomb" protection: a tiny file can describe a 50000×50000 canvas that explodes RAM on decode).

```go
import (
	"image"
	_ "image/gif"  // side-effect import registers the decoder
	_ "image/jpeg"
	_ "image/png"
)

func validateImageConfig(r io.Reader) (w, h int, err error) {
	cfg, _, err := image.DecodeConfig(r) // reads only the header, cheap
	if err != nil {
		return 0, 0, fmt.Errorf("not a decodable image: %w", err)
	}
	if cfg.Width <= 0 || cfg.Height <= 0 || cfg.Width > 12000 || cfg.Height > 12000 {
		return 0, 0, errors.New("image dimensions out of range")
	}
	return cfg.Width, cfg.Height, nil
}
```

### 7.6 Streaming large files & hard size limits **[A]**

For large uploads (videos, archives) don't buffer in RAM. Wrap the body in `http.MaxBytesReader` so an oversized upload is rejected mid-stream rather than after fully buffering, and stream to disk/object-store via `io.Copy`.

```go
func uploadLarge(c *gin.Context) {
	const maxUpload = 200 << 20 // 200 MiB hard ceiling
	// MaxBytesReader returns an error once the limit is exceeded — bounded memory.
	c.Request.Body = http.MaxBytesReader(c.Writer, c.Request.Body, maxUpload)

	fh, err := c.FormFile("file")
	if err != nil {
		c.JSON(413, gin.H{"error": "file too large or missing"}) // 413 Payload Too Large
		return
	}
	src, _ := fh.Open()
	defer src.Close()
	dst, _ := os.Create(filepath.Join("uploads", uuid.NewString()))
	defer dst.Close()
	if _, err := io.Copy(dst, src); err != nil { // streams in 32 KiB chunks; no full buffer
		c.JSON(500, gin.H{"error": "save failed"})
		return
	}
	c.JSON(201, gin.H{"size": fh.Size})
}
```

For a project-wide cap on *all* requests (not just uploads), use the `gin-contrib/size` middleware: `r.Use(size.RequestSizeLimiter(10 << 20))`.

### 7.7 Storing to object storage (S3-compatible) **[A]**

In production, store uploads in object storage (S3, Cloudflare R2, MinIO, Backblaze B2 — all expose an S3 API) rather than local disk, so any instance can serve them and they survive container restarts.

```go
import (
	"github.com/aws/aws-sdk-go-v2/aws"
	"github.com/aws/aws-sdk-go-v2/service/s3"
)

func putObject(ctx context.Context, s3c *s3.Client, bucket, key string, body io.Reader, ct string) error {
	_, err := s3c.PutObject(ctx, &s3.PutObjectInput{
		Bucket:      aws.String(bucket),
		Key:         aws.String(key),
		Body:        body,
		ContentType: aws.String(ct),
	})
	return err
}
```

> **Best practice:** For user-uploaded files behind auth, prefer **pre-signed URLs**: the client uploads directly to object storage using a short-lived signed URL your API generates, so large files never transit your servers. Serve private files the same way (signed GET URLs) instead of proxying bytes.

### 7.8 Serving uploaded files & building URLs **[I]**

```go
// Serve a local directory (development / small apps):
r.Static("/uploads", "./uploads") // GET /uploads/images/abc.jpg

func buildURL(c *gin.Context, name string) string {
	scheme := "http"
	if c.Request.TLS != nil || c.GetHeader("X-Forwarded-Proto") == "https" {
		scheme = "https"
	}
	return fmt.Sprintf("%s://%s/uploads/images/%s", scheme, c.Request.Host, name)
}
```

> **Security recommendation:** Store uploads **outside** any directory containing executable code or templates, serve them from a dedicated path/domain, and set `Content-Disposition: attachment` (and a restrictive `Content-Type`) for user files so a browser won't execute an uploaded `.html`/`.svg` as active content. Consider serving user content from a separate cookieless domain to neutralise stored-XSS.

### 7.9 Upload security checklist **[A]**

- Generate your own filename (UUID + allowlisted extension); never use `fh.Filename`.
- Detect type by magic bytes, not extension or client `Content-Type`.
- Cap size (`MaxBytesReader` / `io.CopyN`) and file count per request.
- Decode images and reject extreme dimensions (decompression bombs).
- Store outside the webroot; serve with safe `Content-Type`/`Content-Disposition`.
- Run uploads through a virus scanner (e.g. ClamAV) for untrusted users.
- Put upload endpoints behind auth + rate limiting (§10, §11, §17).

---

## 8. The Two Folder Structures

This is the headline decision for a project that needs to *scale and stay maintainable*. There is no Go-enforced layout, but two dominant philosophies have emerged. Beginners often pick one accidentally and regret it; choosing deliberately, knowing the trade-offs, is the mark of a senior engineer. We'll lay out the *same* example — a **Users + Orders API** — both ways, then compare.

Both share two non-negotiable Go conventions:

- **`cmd/<app>/main.go`** is the entry point (the *composition root* where everything is wired). Keeping `main` tiny and putting logic in importable packages makes the code testable.
- **`internal/`** is compiler-enforced privacy: packages under `internal/` can only be imported by code rooted at the parent of `internal/`. Put all your application code there so nothing leaks into other modules' public API. `pkg/` is for genuinely reusable libraries you'd be happy for *other* repos to import.

### 8.1 (a) Technical / layer-based structure **[A]**

Here you group files by their **technical role** — all handlers together, all services together, all repositories together. The package boundaries follow the architectural layers (§9).

```
my-api/
├── cmd/
│   └── api/
│       └── main.go                 # composition root: build config, db, wire layers, serve
├── internal/
│   ├── config/
│   │   └── config.go               # typed Config struct, Load(), validate at startup
│   ├── domain/                     # innermost: entities + domain errors, no deps
│   │   ├── user.go                 #   User entity, ErrUserNotFound, ErrEmailTaken
│   │   └── order.go                #   Order entity, ErrOrderNotFound
│   ├── handler/                    # transport layer: all Gin handlers live here
│   │   ├── user_handler.go
│   │   ├── order_handler.go
│   │   └── dto.go                  #   request/response DTOs
│   ├── service/                    # use-case layer: all business logic
│   │   ├── user_service.go
│   │   └── order_service.go
│   ├── repository/                 # data layer: all repository impls
│   │   ├── user_repo.go            #   interface + pgx impl + (often) memory fake
│   │   └── order_repo.go
│   ├── middleware/                 # auth, logging, request-id, recovery, cors, ratelimit
│   │   ├── auth.go
│   │   └── logging.go
│   └── server/
│       └── router.go               # build the gin.Engine, register all routes
├── pkg/                            # optional shared libs (e.g. a generic pagination helper)
├── migrations/                     # golang-migrate .sql files
├── Dockerfile
├── go.mod
└── go.sum
```

**The logic / why:** It mirrors the layered architecture one-to-one, so the dependency rule (handler → service → repository → domain) is visible in the import graph between packages, and `go` can enforce it. Newcomers from Java/.NET/Rails recognise it instantly ("controllers", "services", "models"). It's excellent for **small-to-medium** services and for teaching the layering.

**The cost / when it hurts:** As features multiply, every package becomes a grab-bag. Adding one feature ("add reviews") means touching `handler/`, `service/`, `repository/`, `domain/`, `server/router.go` — five packages for one capability. The packages grow large and *low-cohesion* (a `service` package containing twelve unrelated services). Two engineers working on different features constantly edit the same files (`router.go`, `dto.go`), causing merge friction. There's no compiler barrier stopping the order service from importing user internals, so cross-feature coupling creeps in unnoticed.

### 8.2 (b) Feature / domain-based structure (vertical slice) **[A]**

Here you group files by **business capability** — everything about "users" in one package, everything about "orders" in another. The layers live *inside* each feature package (as files or sub-packages), not as top-level packages.

```
my-api/
├── cmd/
│   └── api/
│       └── main.go                 # composition root: wire features, serve
├── internal/
│   ├── user/                       # EVERYTHING about users in one cohesive package
│   │   ├── user.go                 #   domain entity + domain errors
│   │   ├── dto.go                  #   request/response DTOs + mappers
│   │   ├── service.go              #   use-cases; defines the Repository interface (port)
│   │   ├── repository.go           #   pgx implementation of that interface
│   │   ├── repository_memory.go    #   in-memory fake for tests
│   │   ├── handler.go              #   Gin handlers + RegisterRoutes(rg *gin.RouterGroup)
│   │   └── service_test.go         #   unit tests using the memory repo
│   ├── order/                      # EVERYTHING about orders
│   │   ├── order.go
│   │   ├── dto.go
│   │   ├── service.go
│   │   ├── repository.go
│   │   ├── handler.go
│   │   └── service_test.go
│   └── shared/                     # cross-cutting code used by multiple features
│       ├── middleware/             #   auth, logging, request-id, recovery, cors
│       ├── config/                 #   typed config
│       ├── httpx/                  #   response/error envelope helpers (§15)
│       └── db/                     #   pgxpool setup, migration runner
├── migrations/
├── Dockerfile
├── go.mod
└── go.sum
```

**The logic / why:** Each feature is a **vertical slice** — high cohesion (everything that changes together lives together) and low coupling (features talk to each other only through small, explicit interfaces, not by reaching into each other's files). Adding "reviews" means adding one `internal/review/` package and one line in `main.go` to wire it; you barely touch existing code. Two teams own two packages and rarely collide. You can even define the `Repository` *interface* next to the service that consumes it (Go best practice: "define interfaces where they're used"), keeping the dependency pointing inward without a separate `repository/` package. When a feature outgrows one folder, promote its layers to sub-packages (`internal/order/{domain,service,repo,transport}/`).

**The cost / when it hurts:** The layering is now a *convention inside each package* rather than enforced by the import graph, so discipline matters more. Shared concerns need a home (`shared/`) and it can become a junk drawer if you're not careful — keep only truly cross-cutting things there. For a genuinely tiny app (3 endpoints) the per-feature split is more ceremony than a flat package.

### 8.3 The same handler, both ways **[A]**

Layer-based — the handler imports the *service package*:

```go
// internal/handler/user_handler.go
package handler

import "github.com/you/my-api/internal/service"

type UserHandler struct{ svc *service.UserService }

func NewUserHandler(svc *service.UserService) *UserHandler { return &UserHandler{svc: svc} }
```

Feature-based — the handler and service are the *same package*, so no cross-package import:

```go
// internal/user/handler.go
package user

type Handler struct{ svc *Service } // Service is in the same package

func NewHandler(svc *Service) *Handler { return &Handler{svc: svc} }

// RegisterRoutes keeps routing co-located with the feature it serves.
func (h *Handler) RegisterRoutes(rg *gin.RouterGroup) {
	rg.POST("/users", h.Create)
	rg.GET("/users/:id", h.GetByID)
}
```

### 8.4 Comparison & recommendation **[A]**

| Dimension | (a) Layer-based | (b) Feature-based |
|---|---|---|
| Cohesion | Low (a package per layer, many features) | High (a package per feature) |
| Cost of adding a feature | Edit ~5 packages | Add 1 package + 1 wire line |
| Team scaling / merge conflicts | More (shared `router.go`, `dto.go`) | Less (one team per package) |
| Cross-feature coupling | Easy to leak; not enforced | Forced through explicit interfaces |
| Where interfaces are defined | Often a separate `repository/` pkg | Next to the consumer (idiomatic Go) |
| Learning curve | Familiar to MVC folks | Needs discipline on layering-within-package |
| Best for | Small/medium services; teaching layering | Medium/large services; multiple teams; long-lived code |

> **Recommendation:** For anything you expect to **grow or be maintained by more than one or two people, use the feature/domain-based (vertical slice) structure** — it is the Go community's prevailing recommendation for scalable services precisely because it keeps coupling low as the system grows. Start *flat* if the app is tiny, and refactor into feature packages the moment you have a second feature. Use the layer-based structure only for small services or when a team strongly prefers the familiar MVC shape. Whichever you pick, keep `main` as the only place that knows concrete types (§9.4), and keep `gin` out of the service/domain layers.

The worked example in §20 uses the **feature-based** structure.

---

## 9. Architecture & Design Patterns for Scale

A folder structure is just where files live; *architecture* is how dependencies point and how responsibilities are split. This section applies the patterns from `GO_LANG_AND_PATTERNS_GUIDE.md` specifically to a Gin backend. Each is introduced by the problem it solves, then shown in code.

### 9.1 Clean / layered architecture & the dependency rule **[A]**

**The problem:** as a backend grows, the lethal failure mode is *coupling* — handlers that run raw SQL, business rules smeared across handlers, a DB swap that touches a hundred files, and tests that need a live database. Everything depends on everything, so nothing changes in isolation.

**The fix — layers + one rule.** Split the code into concentric layers and enforce that **dependencies point inward**:

1. **Domain / entities** (innermost): core types and pure business rules (`User`, `Order`). Imports *nothing* — no Gin, no DB, no HTTP.
2. **Service / use-cases**: orchestrates operations (`RegisterUser`, `PlaceOrder`). Depends on the domain and on **interfaces (ports)** for what it needs (a `Repository`, an `Emailer`) — never on concrete infrastructure.
3. **Adapters / infrastructure** (outer): concrete implementations of those ports — a pgx repository, an SMTP client. They depend inward (they implement interfaces the service defined).
4. **Transport / delivery** (outermost): Gin handlers. Translate HTTP ↔ service calls. The *only* layer that imports `gin`.

```
┌──────────────────────────────────────────────┐
│ Transport (Gin handlers)  — imports gin       │  outer
│  ┌─────────────────────────────────────────┐ │
│  │ Service / use-cases (no gin, no SQL)      │ │
│  │  ┌────────────────────────────────────┐  │ │
│  │  │ Domain (entities, rules) — no deps   │  │ │  inner
│  │  └────────────────────────────────────┘  │ │
│  │  defines ports (interfaces) ───────────► │ │
│  └─────────────────────────────────────────┘ │
│ Adapters (pgx repo, SMTP) implement the ports │
└──────────────────────────────────────────────┘
            dependencies point INWARD
```

Go's **implicit interface satisfaction** makes this natural: the service defines a `Repository` interface without importing any DB package, and the pgx package implements it without importing the service. Neither knows the other's concrete type.

### 9.2 Handler → Service → Repository **[A]**

The canonical three-layer flow for one request. Each layer has one job and depends only on the layer beneath it (through an interface for the repo):

- **Handler:** parse/validate the request (DTO), call the service, map the result/ error to an HTTP response. *No business logic, no SQL.*
- **Service:** business rules and orchestration. *No HTTP types, no SQL.*
- **Repository:** persistence. *No business rules.*

```go
// transport: handler depends on the service interface, speaks HTTP/DTO only.
func (h *Handler) Create(c *gin.Context) {
	var req CreateUserRequest
	if err := c.ShouldBindJSON(&req); err != nil {
		respondValidationError(c, err)
		return
	}
	u, err := h.svc.Register(c.Request.Context(), req.Email, req.Name, req.Password)
	if err != nil {
		respondDomainError(c, err) // maps domain error → status (§15)
		return
	}
	c.JSON(http.StatusCreated, toUserResponse(u))
}
```

### 9.3 Repository pattern (interface + pgx + in-memory fake) **[A]**

**Why:** business logic must not know *how* data is stored. The repository is an interface in *domain terms* (`Save`, `FindByID`) defined in the **consumer's** (service's) package so the dependency points inward. You get a pgx impl for production and a memory impl for fast tests, interchangeable because they satisfy the same interface.

```go
// internal/user/user.go — domain + the port. No DB import here.
package user

import (
	"context"
	"errors"
	"time"
)

type User struct {
	ID, Email, Name, PasswordHash, Role string
	CreatedAt                           time.Time
}

var (
	ErrNotFound   = errors.New("user not found")
	ErrEmailTaken = errors.New("email already registered")
)

// Repository is the PORT — defined where it's consumed (the service package).
type Repository interface {
	Save(ctx context.Context, u User) error
	FindByID(ctx context.Context, id string) (User, error)
	FindByEmail(ctx context.Context, email string) (User, error)
}
```

```go
// internal/user/repository.go — pgx adapter (production).
package user

import (
	"context"
	"errors"

	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgxpool"
)

type PostgresRepo struct{ pool *pgxpool.Pool }

func NewPostgresRepo(pool *pgxpool.Pool) *PostgresRepo { return &PostgresRepo{pool: pool} }

func (r *PostgresRepo) FindByID(ctx context.Context, id string) (User, error) {
	var u User
	err := r.pool.QueryRow(ctx,
		`SELECT id, email, name, password_hash, role, created_at FROM users WHERE id=$1`, id).
		Scan(&u.ID, &u.Email, &u.Name, &u.PasswordHash, &u.Role, &u.CreatedAt)
	if errors.Is(err, pgx.ErrNoRows) {
		return User{}, ErrNotFound // translate DB error → DOMAIN error (anti-corruption)
	}
	return u, err
}
// ...Save, FindByEmail similarly, all using parameterised queries.
```

```go
// internal/user/repository_memory.go — fake for unit tests (no DB).
package user

import (
	"context"
	"sync"
)

type MemoryRepo struct {
	mu   sync.RWMutex
	byID map[string]User
}

func NewMemoryRepo() *MemoryRepo { return &MemoryRepo{byID: map[string]User{}} }

func (r *MemoryRepo) Save(ctx context.Context, u User) error {
	r.mu.Lock()
	defer r.mu.Unlock()
	for _, e := range r.byID {
		if e.Email == u.Email && e.ID != u.ID {
			return ErrEmailTaken
		}
	}
	r.byID[u.ID] = u
	return nil
}
func (r *MemoryRepo) FindByID(ctx context.Context, id string) (User, error) {
	r.mu.RLock()
	defer r.mu.RUnlock()
	if u, ok := r.byID[id]; ok {
		return u, nil
	}
	return User{}, ErrNotFound
}
func (r *MemoryRepo) FindByEmail(ctx context.Context, email string) (User, error) {
	r.mu.RLock()
	defer r.mu.RUnlock()
	for _, u := range r.byID {
		if u.Email == email {
			return u, nil
		}
	}
	return User{}, ErrNotFound
}
```

> **Trade-off:** the indirection is overkill for a tiny CRUD app — add it when you have a real testing need or a second implementation. Don't leak storage concepts (`*sql.Tx`) into the interface; keep it in domain terms (transactions: §12.3).

### 9.4 Dependency injection (constructor injection + wiring in main) **[A]**

DI in Go needs no framework: a component *receives* its dependencies (as interfaces) instead of constructing them. The service takes a `Repository` interface; `main` — the **composition root** — decides which concrete implementation to pass. To switch memory→Postgres you change one line in `main`.

```go
// internal/user/service.go — depends on the interface, injected via the constructor.
package user

type Service struct {
	repo   Repository // interface; concrete type chosen by the caller
	hasher PasswordHasher
}

func NewService(repo Repository, hasher PasswordHasher) *Service {
	return &Service{repo: repo, hasher: hasher}
}
```

```go
// cmd/api/main.go — the ONE place that names concrete types.
pool := mustConnectDB(cfg)              // *pgxpool.Pool
userRepo := user.NewPostgresRepo(pool)  // ← swap to user.NewMemoryRepo() for local dev
userSvc := user.NewService(userRepo, argon2Hasher{})
userHandler := user.NewHandler(userSvc)
```

> **wire / fx note:** For large object graphs, `google/wire` *generates* the wiring at compile time (no runtime reflection) and `uber-go/fx` does it at runtime with lifecycle hooks. For most apps, **manual constructor injection is the recommended default** — explicit, debuggable, framework-free. Reach for `wire` only when the boilerplate genuinely hurts.

### 9.5 Factory constructors & functional options **[A]**

Go has no constructors, so the convention is a **factory function** `NewX` that validates inputs and returns a ready-to-use value (or an error). When a constructor has many *optional* parameters with defaults, use **functional options** — the idiomatic Go config pattern (you'll see it in gRPC, the AWS SDK, etc.).

```go
type Server struct {
	addr    string
	timeout time.Duration
	logger  *slog.Logger
}

type Option func(*Server) // an option mutates the server during construction

func WithTimeout(d time.Duration) Option { return func(s *Server) { s.timeout = d } }
func WithLogger(l *slog.Logger) Option   { return func(s *Server) { s.logger = l } }

func NewServer(addr string, opts ...Option) *Server {
	s := &Server{addr: addr, timeout: 30 * time.Second, logger: slog.Default()} // defaults
	for _, opt := range opts { // later options override earlier ones
		opt(s)
	}
	return s
}
// NewServer(":8080", WithTimeout(5*time.Second)) — pass only what you care about.
```

### 9.6 Adapter / anti-corruption layer for external clients **[A]**

When you call a third-party SDK (payments, email, cloud), wrap it behind an interface *you* define in *your* terms — the **anti-corruption layer**. Your service depends on `Emailer`, never on the vendor's types; the vendor SDK is imported in exactly one file, so swapping providers or faking it in tests touches nothing else.

```go
type Emailer interface {
	Send(ctx context.Context, to, subject, body string) error
}

type sendgridAdapter struct {
	client *vendor.Client
	from   string
}

func NewSendgridEmailer(key, from string) Emailer { // returns the INTERFACE
	return &sendgridAdapter{client: vendor.NewClient(key), from: from}
}

func (a *sendgridAdapter) Send(ctx context.Context, to, subj, body string) error {
	resp, err := a.client.SendWithContext(ctx, vendor.NewMessage(a.from, to, subj, body))
	if err != nil {
		return fmt.Errorf("sendgrid: %w", err) // wrap vendor errors → your callers see plain error
	}
	if resp.StatusCode >= 400 {
		return fmt.Errorf("sendgrid rejected: %d", resp.StatusCode)
	}
	return nil
}
```

### 9.7 Middleware as the decorator pattern & DTO mapping **[A]**

Gin **middleware** *is* the decorator pattern: a function that wraps the handler chain to add behaviour (logging, auth, recovery) without changing the handlers (§10). **DTO mapping** (§5.6) is the adapter pattern applied to data: explicit `toXResponse` / `req.toDomain()` functions keep the wire shape decoupled from the domain shape. Keep mappers tiny, total, and in the transport layer.

### 9.8 Service/use-case layer & domain errors → HTTP **[A]**

The service returns **domain errors** (`user.ErrNotFound`, `user.ErrEmailTaken`) — it knows nothing about HTTP status codes. The *transport* layer maps those to statuses in one place (§15), so the mapping is consistent and the business logic stays clean and portable (the same service could back a gRPC or CLI front-end).

```go
func (s *Service) Register(ctx context.Context, email, name, pw string) (User, error) {
	if _, err := s.repo.FindByEmail(ctx, email); err == nil {
		return User{}, ErrEmailTaken                 // business rule → domain error
	} else if !errors.Is(err, ErrNotFound) {
		return User{}, err                           // a real failure, not "absent"
	}
	hash, err := s.hasher.Hash(pw)
	if err != nil {
		return User{}, err
	}
	u := User{ID: uuid.NewString(), Email: email, Name: name, PasswordHash: hash,
		Role: "user", CreatedAt: time.Now().UTC()}
	if err := s.repo.Save(ctx, u); err != nil {
		return User{}, err
	}
	return u, nil
}
```

---

## 10. Middleware In Depth

Middleware is a `gin.HandlerFunc` (same type as a handler) inserted into the chain. It can run code before the handler (on the way in), call `c.Next()` to proceed, and run code after (on the way out). It's how you implement every cross-cutting concern once and apply it everywhere.

### 10.1 Anatomy, ordering & where to register **[I]**

```
Request
  → recovery (outermost: must wrap everything to catch panics)
    → requestID (generate/propagate correlation ID)
      → logger (start timer; logs on the way out)
        → cors / security headers
          → rate limiter (cheap rejection before expensive work)
            → auth (only on protected groups)
              → YOUR HANDLER
            ← auth
          ← rate limiter
        ← logger (writes the access log line with status + latency)
  ← recovery
Response
```

**Order matters.** Register globally with `r.Use(...)` in the order above; recovery and request-ID first (so even failures are logged with an ID), auth on the protected *group* only (so public routes stay open). The "after `c.Next()`" code runs in reverse registration order.

```go
r := gin.New()
r.Use(middleware.Recovery(logger))   // 1. catch panics → 500 (outermost)
r.Use(middleware.RequestID())        // 2. correlation ID for every request
r.Use(middleware.Logger(logger))     // 3. structured access log
r.Use(middleware.SecureHeaders())    // 4. security response headers
r.Use(cors.New(cfg.CORSConfig()))    // 5. CORS
r.Use(middleware.RateLimit(rl))      // 6. global rate limit

public := r.Group("/api/v1")
public.POST("/auth/login", login)    // no auth needed

protected := r.Group("/api/v1")
protected.Use(middleware.Auth(jwtMgr)) // 7. auth ONLY here
protected.GET("/me", me)
```

### 10.2 Request ID / correlation ID **[I]**

Every request gets a unique ID, surfaced in logs and echoed back so a client (or you) can correlate a user-reported error with server logs. Reuse an inbound `X-Request-ID` if a trusted upstream set one.

```go
const HeaderRequestID = "X-Request-ID"

func RequestID() gin.HandlerFunc {
	return func(c *gin.Context) {
		id := c.GetHeader(HeaderRequestID)
		if id == "" {
			id = uuid.NewString()
		}
		c.Set("request_id", id)
		c.Header(HeaderRequestID, id) // echo back to the client
		c.Next()
	}
}
```

### 10.3 Structured logging middleware (slog) **[A]**

Use Go's stdlib `log/slog` (or `zap` for max performance). Log *structured* fields (method, path, status, latency, request_id) so logs are queryable in aggregation tools — never `fmt.Printf` strings.

```go
func Logger(base *slog.Logger) gin.HandlerFunc {
	return func(c *gin.Context) {
		start := time.Now()
		path := c.Request.URL.Path
		c.Next() // run the handler

		base.LogAttrs(c.Request.Context(), slog.LevelInfo, "http_request",
			slog.String("method", c.Request.Method),
			slog.String("path", path),
			slog.Int("status", c.Writer.Status()),
			slog.Int("bytes", c.Writer.Size()),
			slog.Duration("latency", time.Since(start)),
			slog.String("client_ip", c.ClientIP()),
			slog.String("request_id", c.GetString("request_id")),
		)
	}
}
```

> **Security recommendation:** Never log secrets, full `Authorization` headers, passwords, tokens, or full request bodies. Log identifiers, not payloads (§17).

### 10.4 Recovery (panic → 500) **[A]**

`gin.Recovery()` works, but a custom one logs structured details with the request ID and returns your error envelope. A panic in any handler must never crash the process.

```go
func Recovery(logger *slog.Logger) gin.HandlerFunc {
	return func(c *gin.Context) {
		defer func() {
			if rec := recover(); rec != nil {
				logger.Error("panic recovered",
					slog.Any("panic", rec),
					slog.String("request_id", c.GetString("request_id")),
					slog.String("stack", string(debug.Stack())))
				// Generic message only — never leak the panic/stack to the client.
				c.AbortWithStatusJSON(http.StatusInternalServerError, ErrorResponse{
					Error: ErrorBody{Code: "internal_error", Message: "internal server error",
						TraceID: c.GetString("request_id")},
				})
			}
		}()
		c.Next()
	}
}
```

### 10.5 CORS **[I]**

CORS controls which browser *origins* may call your API. Use `gin-contrib/cors` and configure it **explicitly** in production — never `AllowAllOrigins` with `AllowCredentials`.

```go
import "github.com/gin-contrib/cors"

r.Use(cors.New(cors.Config{
	AllowOrigins:     []string{"https://app.example.com"}, // explicit allowlist
	AllowMethods:     []string{"GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"},
	AllowHeaders:     []string{"Origin", "Content-Type", "Authorization", "X-Request-ID"},
	ExposeHeaders:    []string{"X-Request-ID"},
	AllowCredentials: true,            // allows cookies/Authorization; requires non-wildcard origin
	MaxAge:           12 * time.Hour,  // cache the preflight
}))
```

> **Security recommendation:** `AllowCredentials: true` is incompatible with `AllowOrigins: ["*"]` (the browser rejects it, and allowing both would be a serious CSRF risk). List exact origins. Use `AllowOriginFunc` for dynamic subdomain checks, but validate strictly.

### 10.6 Gzip, rate limiting, timeouts, body-size **[A]**

```go
import (
	"github.com/gin-contrib/gzip"
	"golang.org/x/time/rate"
)

// Gzip compression (skip for already-compressed assets/uploads):
r.Use(gzip.Gzip(gzip.DefaultCompression))

// Token-bucket rate limit (per instance). For multi-instance, back it with Redis (§16.6).
func RateLimit(perSec float64, burst int) gin.HandlerFunc {
	limiters := &sync.Map{} // clientIP → *rate.Limiter
	return func(c *gin.Context) {
		ip := c.ClientIP()
		lAny, _ := limiters.LoadOrStore(ip, rate.NewLimiter(rate.Limit(perSec), burst))
		if !lAny.(*rate.Limiter).Allow() {
			c.Header("Retry-After", "1")
			c.AbortWithStatusJSON(http.StatusTooManyRequests, ErrorResponse{
				Error: ErrorBody{Code: "rate_limited", Message: "too many requests"}})
			return
		}
		c.Next()
	}
}

// Per-request timeout: derive a context with a deadline and pass it down.
func Timeout(d time.Duration) gin.HandlerFunc {
	return func(c *gin.Context) {
		ctx, cancel := context.WithTimeout(c.Request.Context(), d)
		defer cancel()
		c.Request = c.Request.WithContext(ctx) // handlers using c.Request.Context() honour it
		c.Next()
	}
}

// Hard body-size limit for ALL requests:
import "github.com/gin-contrib/size"
r.Use(size.RequestSizeLimiter(10 << 20)) // 10 MiB
```

> **Best practice:** A per-instance limiter caps *that pod*; with N pods the effective limit is N×. For a true global limit, use a distributed limiter (Redis token bucket). Always add `Retry-After` on 429.

---

## 11. Authentication & Authorization

**Authentication** = who are you (proving identity). **Authorization** = what may you do (permissions). Keep them as separate middleware so you can mix them (some routes need auth + admin; some only auth). This section covers JWTs, RBAC, password hashing, API keys, and cookie vs token. For token/hash internals, see `JWT_AUTH_ARGON2_GUIDE.md`.

### 11.1 Session vs token; cookie vs Authorization header **[A]**

- **Sessions** (server-side state, a `session_id` cookie): the server stores session data; the cookie is just a key. Easy to revoke (delete the session), but requires shared session storage (Redis) across instances — slightly less "stateless".
- **Tokens (JWT)**: the token *is* the state (signed claims); the server stores nothing. Perfectly stateless and horizontally scalable, but **hard to revoke** before expiry — which is why you use *short-lived* access tokens plus refresh tokens (§11.3).
- **Cookie vs header:** Browser apps benefit from `HttpOnly; Secure; SameSite` cookies (immune to JS/XSS token theft, but need CSRF protection). Native/mobile/SPA-to-API clients typically send `Authorization: Bearer <token>`. Choose per client type.

### 11.2 Password hashing (Argon2id / bcrypt) **[A]**

Never store plaintext or fast hashes (MD5/SHA). Use a slow, salted, memory-hard KDF: **Argon2id** (preferred in 2026) or **bcrypt**. Define a small interface so the service doesn't depend on the algorithm.

```go
type PasswordHasher interface {
	Hash(plain string) (string, error)
	Verify(plain, encoded string) (bool, error)
}

// bcrypt is simplest (cost 12+). Argon2id is stronger — see JWT_AUTH_ARGON2_GUIDE.md.
import "golang.org/x/crypto/bcrypt"

type BcryptHasher struct{ cost int }

func (h BcryptHasher) Hash(plain string) (string, error) {
	b, err := bcrypt.GenerateFromPassword([]byte(plain), h.cost) // salt is embedded in output
	return string(b), err
}
func (h BcryptHasher) Verify(plain, encoded string) (bool, error) {
	err := bcrypt.CompareHashAndPassword([]byte(encoded), []byte(plain)) // constant-time
	if errors.Is(err, bcrypt.ErrMismatchedHashAndPassword) {
		return false, nil
	}
	return err == nil, err
}
```

> **Security recommendation:** On a failed login return the *same* generic "invalid credentials" whether the email or the password was wrong (don't reveal which), and run the hash verification even for unknown emails (compare against a dummy hash) to avoid a timing/enumeration oracle. Rate-limit login attempts (§17).

### 11.3 JWT access + refresh tokens **[A]**

The pattern: a **short-lived access token** (5–15 min) sent on every request, and a **long-lived refresh token** (days/weeks, stored server-side or as a hashed value) used only to mint new access tokens. Short access tokens bound the damage if one leaks; the refresh token can be revoked (it's tracked).

```go
import "github.com/golang-jwt/jwt/v5"

type Claims struct {
	UserID string `json:"uid"`
	Role   string `json:"role"`
	jwt.RegisteredClaims
}

type JWTManager struct {
	secret    []byte
	accessTTL time.Duration
	issuer    string
}

func (m *JWTManager) NewAccessToken(userID, role string) (string, error) {
	now := time.Now()
	claims := Claims{
		UserID: userID, Role: role,
		RegisteredClaims: jwt.RegisteredClaims{
			Issuer:    m.issuer,
			Subject:   userID,
			IssuedAt:  jwt.NewNumericDate(now),
			ExpiresAt: jwt.NewNumericDate(now.Add(m.accessTTL)),
		},
	}
	return jwt.NewWithClaims(jwt.SigningMethodHS256, claims).SignedString(m.secret)
}

func (m *JWTManager) Parse(tokenStr string) (*Claims, error) {
	claims := &Claims{}
	_, err := jwt.ParseWithClaims(tokenStr, claims, func(t *jwt.Token) (any, error) {
		// CRITICAL: pin the algorithm. Reject anything that isn't HMAC to defeat
		// the classic "alg: none" and RS256→HS256 confusion attacks.
		if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
			return nil, errors.New("unexpected signing method")
		}
		return m.secret, nil
	}, jwt.WithIssuer(m.issuer), jwt.WithExpirationRequired())
	if err != nil {
		return nil, err
	}
	return claims, nil
}
```

> **Security recommendation:** Always pin the expected signing algorithm in the keyfunc (above). Load the secret from config/secret-manager, never hardcode. Require expiry. Keep access-token TTL short. On logout/compromise, revoke the refresh token (delete its server record / add its jti to a denylist).

### 11.4 Auth middleware **[A]**

```go
func Auth(jwtMgr *JWTManager) gin.HandlerFunc {
	return func(c *gin.Context) {
		h := c.GetHeader("Authorization")
		if !strings.HasPrefix(h, "Bearer ") {
			c.AbortWithStatusJSON(401, ErrorResponse{Error: ErrorBody{
				Code: "unauthenticated", Message: "missing bearer token"}})
			return
		}
		claims, err := jwtMgr.Parse(strings.TrimPrefix(h, "Bearer "))
		if err != nil {
			c.AbortWithStatusJSON(401, ErrorResponse{Error: ErrorBody{
				Code: "unauthenticated", Message: "invalid or expired token"}})
			return
		}
		c.Set("userID", claims.UserID) // hand identity to downstream handlers
		c.Set("role", claims.Role)
		c.Next()
	}
}
```

### 11.5 RBAC (role-based authorization) **[A]**

Authorization middleware runs *after* auth and checks the role/permission stored on the context. Keep it data-driven for many roles.

```go
// Require any of the allowed roles. Chain after Auth().
func RequireRole(allowed ...string) gin.HandlerFunc {
	set := make(map[string]struct{}, len(allowed))
	for _, r := range allowed {
		set[r] = struct{}{}
	}
	return func(c *gin.Context) {
		if _, ok := set[c.GetString("role")]; !ok {
			c.AbortWithStatusJSON(403, ErrorResponse{Error: ErrorBody{
				Code: "forbidden", Message: "insufficient permissions"}})
			return
		}
		c.Next()
	}
}
// admin := protected.Group("/admin"); admin.Use(RequireRole("admin"))
```

> **Best practice:** For row-level checks ("can THIS user edit THIS order?"), middleware isn't enough — enforce ownership in the *service* layer where you have the resource. RBAC middleware handles coarse role gates; the service handles fine-grained, data-dependent authorization.

### 11.6 API keys & OAuth (notes) **[I]**

- **API keys** (for server-to-server / machine clients): issue a random high-entropy key, store only its **hash**, and look it up in middleware. Treat like a password — never log it, allow rotation, scope it.
- **OAuth2 / OIDC** (delegated login via Google/GitHub/etc.): your API receives an OAuth/OIDC token; validate it against the provider's JWKS and map the verified identity to a local user. Use a vetted library (`golang.org/x/oauth2`, `coreos/go-oidc`); don't hand-roll the flow.

### 11.7 Secure cookie flags **[I]**

If you do use cookies (refresh tokens, sessions), set the protective flags:

```go
c.SetSameSite(http.SameSiteStrictMode) // CSRF defence
c.SetCookie("refresh_token", token,
	int((7 * 24 * time.Hour).Seconds()), // maxAge seconds
	"/auth", "", // path scoped to the refresh endpoint; domain default
	true,        // Secure: HTTPS only
	true)        // HttpOnly: invisible to JavaScript (XSS theft defence)
```

---

## 12. Database Layer

For relational data use **pgx** (the high-performance Postgres driver/toolkit) directly, or **GORM** (an ORM) when you want less boilerplate. This guide shows pgx; see `POSTGRESQL_GUIDE.md` and `RELATIONAL_DB_DESIGN_GUIDE.md` for SQL/schema depth and `GO_GORM_ENT_GUIDE.md` for the ORM route.

### 12.1 Connection pooling **[A]**

A web server handles many concurrent requests; each DB query needs a connection. Opening one per request is far too slow, so you use a **pool** (`pgxpool`) of reusable connections sized to your DB's capacity. Tune the pool — too small and requests queue; too large and you exhaust Postgres's `max_connections`.

```go
import "github.com/jackc/pgx/v5/pgxpool"

func NewPool(ctx context.Context, dsn string) (*pgxpool.Pool, error) {
	cfg, err := pgxpool.ParseConfig(dsn)
	if err != nil {
		return nil, err
	}
	cfg.MaxConns = 20                       // cap concurrent connections (≤ Postgres max)
	cfg.MinConns = 2                        // keep a few warm
	cfg.MaxConnLifetime = time.Hour         // recycle to avoid stale conns
	cfg.MaxConnIdleTime = 30 * time.Minute
	cfg.HealthCheckPeriod = time.Minute

	pool, err := pgxpool.NewWithConfig(ctx, cfg)
	if err != nil {
		return nil, err
	}
	// Fail fast at startup if the DB is unreachable:
	pingCtx, cancel := context.WithTimeout(ctx, 5*time.Second)
	defer cancel()
	if err := pool.Ping(pingCtx); err != nil {
		return nil, fmt.Errorf("db ping: %w", err)
	}
	return pool, nil
}
```

### 12.2 Query patterns & SQL-injection safety **[A]**

**Always use parameterised queries** (`$1`, `$2`). The driver sends the SQL and the values separately, so user input can never be interpreted as SQL — this is the single most important defence against SQL injection. **Never** build queries with `fmt.Sprintf`/string concatenation of user input.

```go
// SAFE — parameterised. Values are bound, not interpolated.
row := pool.QueryRow(ctx, `SELECT id, email FROM users WHERE email = $1`, email)

// Reading many rows with pgx.CollectRows + RowToStructByName (pgx v5):
rows, _ := pool.Query(ctx, `SELECT id, total FROM orders WHERE user_id=$1 ORDER BY id LIMIT $2`, uid, limit)
orders, err := pgx.CollectRows(rows, pgx.RowToStructByName[Order])

// NEVER do this:
// pool.Query(ctx, "SELECT * FROM users WHERE email = '" + email + "'") // INJECTION
```

> **Note on prepared statements:** pgx automatically prepares and caches statements per connection, so you get prepared-statement performance and safety without managing them by hand.

### 12.3 Transactions (passing tx through the service) **[A]**

A transaction groups multiple writes so they all succeed or all roll back. The classic problem: a use-case spans two repositories (debit order, credit ledger) that must commit together. Don't leak `pgx.Tx` into your domain interfaces; instead expose a **transactor** that runs a function within a transaction, and have repositories accept a query-runner interface satisfied by both the pool and a tx.

```go
// A minimal interface satisfied by both *pgxpool.Pool and pgx.Tx:
type DB interface {
	Exec(ctx context.Context, sql string, args ...any) (pgconn.CommandTag, error)
	QueryRow(ctx context.Context, sql string, args ...any) pgx.Row
	Query(ctx context.Context, sql string, args ...any) (pgx.Rows, error)
}

// Transactor runs fn inside a transaction, committing or rolling back automatically.
type Transactor struct{ pool *pgxpool.Pool }

func (t *Transactor) WithinTx(ctx context.Context, fn func(tx pgx.Tx) error) error {
	tx, err := t.pool.Begin(ctx)
	if err != nil {
		return err
	}
	defer tx.Rollback(ctx) // no-op if already committed; safety net on early return/panic
	if err := fn(tx); err != nil {
		return err // rollback via defer
	}
	return tx.Commit(ctx)
}

// Service use-case spanning two repos atomically:
func (s *OrderService) Place(ctx context.Context, o Order) error {
	return s.tx.WithinTx(ctx, func(tx pgx.Tx) error {
		if err := s.orders.SaveTx(ctx, tx, o); err != nil {
			return err
		}
		return s.ledger.DebitTx(ctx, tx, o.UserID, o.Total) // both commit together
	})
}
```

### 12.4 Migrations (golang-migrate) **[A]**

Schema changes must be versioned and repeatable — never hand-edited in production. Use `golang-migrate`: numbered `up`/`down` SQL files applied in order, tracked in a `schema_migrations` table.

```
migrations/
  0001_create_users.up.sql     /  0001_create_users.down.sql
  0002_create_orders.up.sql    /  0002_create_orders.down.sql
```

```bash
# CLI:
migrate -path migrations -database "$DATABASE_URL" up
migrate -path migrations -database "$DATABASE_URL" down 1
```

```go
// Or run at startup from Go (golang-migrate library):
import (
	"github.com/golang-migrate/migrate/v4"
	_ "github.com/golang-migrate/migrate/v4/database/pgx/v5"
	_ "github.com/golang-migrate/migrate/v4/source/file"
)

func runMigrations(dsn string) error {
	m, err := migrate.New("file://migrations", dsn)
	if err != nil {
		return err
	}
	if err := m.Up(); err != nil && !errors.Is(err, migrate.ErrNoChange) {
		return err
	}
	return nil
}
```

### 12.5 Avoiding N+1 queries **[A]**

The **N+1 problem**: you fetch N orders (1 query), then loop and fetch each order's items (N more queries) — N+1 round trips, catastrophic under load. Fix by fetching related data in **one** query: a JOIN, or a single `WHERE id = ANY($1)` over the collected IDs.

```go
// BAD — N+1:
for _, o := range orders {
	o.Items, _ = itemRepo.FindByOrder(ctx, o.ID) // one query PER order
}

// GOOD — one batched query, then group in memory:
ids := make([]string, len(orders))
for i, o := range orders {
	ids[i] = o.ID
}
rows, _ := pool.Query(ctx, `SELECT order_id, sku, qty FROM items WHERE order_id = ANY($1)`, ids)
// scan rows, then bucket items by order_id into the corresponding order.
```

> **Best practice:** Always pass `c.Request.Context()` into every DB call so a client disconnect or request timeout cancels the query instead of holding a connection. Add appropriate indexes (see `RELATIONAL_DB_DESIGN_GUIDE.md`) — a missing index turns a fast query into a table scan that pools can't save you from.

---

## 13. Configuration & Secrets

Configuration is everything that varies between environments (dev/staging/prod): ports, DB DSNs, secrets, timeouts, feature flags. Get this wrong and you'll hardcode a secret into a binary or ship debug mode to production.

### 13.1 The 12-factor approach — config in the environment **[I]**

The **12-factor** rule: store config in **environment variables**, not in code or committed files. The same compiled binary then runs in any environment by changing env vars only — no rebuild. Locally you can use a `.env` file (loaded with `godotenv`), but `.env` is **gitignored** and never committed.

### 13.2 Typed config struct + validation at startup **[I/A]**

Parse all config into one **typed struct** once at startup, validate it, and **fail fast** (exit) if anything required is missing or malformed. This turns "mysterious 3am nil DSN" into "clear startup error". Below uses plain `os` parsing; **Viper** adds files/flags/precedence if you outgrow it.

```go
package config

import (
	"fmt"
	"os"
	"strconv"
	"time"
)

type Config struct {
	Env             string        // "development" | "production"
	Addr            string        // ":8080"
	GinMode         string        // "release"
	DatabaseURL     string        // postgres DSN (secret)
	JWTSecret       string        // signing secret (secret)
	AccessTokenTTL  time.Duration
	AllowedOrigins  []string
	MaxUploadBytes  int64
	ShutdownTimeout time.Duration
}

func Load() (*Config, error) {
	c := &Config{
		Env:             getenv("APP_ENV", "development"),
		Addr:            getenv("ADDR", ":8080"),
		GinMode:         getenv("GIN_MODE", "debug"),
		DatabaseURL:     os.Getenv("DATABASE_URL"),
		JWTSecret:       os.Getenv("JWT_SECRET"),
		AccessTokenTTL:  getdur("ACCESS_TOKEN_TTL", 15*time.Minute),
		MaxUploadBytes:  getint64("MAX_UPLOAD_BYTES", 8<<20),
		ShutdownTimeout: getdur("SHUTDOWN_TIMEOUT", 15*time.Second),
	}
	// Validate REQUIRED secrets — fail fast, do not start half-configured.
	if c.DatabaseURL == "" {
		return nil, fmt.Errorf("DATABASE_URL is required")
	}
	if len(c.JWTSecret) < 32 {
		return nil, fmt.Errorf("JWT_SECRET must be at least 32 bytes")
	}
	return c, nil
}

func getenv(k, def string) string {
	if v := os.Getenv(k); v != "" {
		return v
	}
	return def
}
func getdur(k string, def time.Duration) time.Duration {
	if v := os.Getenv(k); v != "" {
		if d, err := time.ParseDuration(v); err == nil {
			return d
		}
	}
	return def
}
func getint64(k string, def int64) int64 {
	if v := os.Getenv(k); v != "" {
		if n, err := strconv.ParseInt(v, 10, 64); err == nil {
			return n
		}
	}
	return def
}
```

### 13.3 Secrets handling **[A]**

> **Security recommendation:** Never commit secrets (add `.env` to `.gitignore`). In production, inject secrets via your platform's secret manager (Kubernetes Secrets, AWS Secrets Manager, Vault, Doppler) as env vars or mounted files — not baked into the image. Rotate secrets periodically. Don't log config that contains secrets; if you log the loaded config for diagnostics, **redact** secret fields. Keep secrets out of error messages returned to clients.

---

## 14. Observability

Observability is your ability to answer "what is the system doing and why" in production. The three pillars are **logs**, **metrics**, and **traces**, tied together by a **correlation ID**.

### 14.1 Structured logging (slog / zap) **[A]**

Use `log/slog` (stdlib, 1.21+) with a JSON handler in production so logs are machine-parseable; `zap` if you need the absolute lowest allocation. Attach the request ID to every log line (§10.3) so all logs for one request can be grouped.

```go
func newLogger(env string) *slog.Logger {
	var h slog.Handler
	if env == "production" {
		h = slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelInfo})
	} else {
		h = slog.NewTextHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelDebug})
	}
	return slog.New(h)
}
```

### 14.2 Metrics (Prometheus) **[A]**

Metrics are aggregate numbers (request counts, latencies, error rates) scraped by Prometheus and graphed in Grafana. Expose `/metrics`; record per-request counters and a latency histogram in middleware.

```go
import (
	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
	httpRequests = prometheus.NewCounterVec(
		prometheus.CounterOpts{Name: "http_requests_total"},
		[]string{"method", "path", "status"})
	httpLatency = prometheus.NewHistogramVec(
		prometheus.HistogramOpts{Name: "http_request_duration_seconds",
			Buckets: prometheus.DefBuckets}, []string{"method", "path"})
)

func init() { prometheus.MustRegister(httpRequests, httpLatency) }

func Metrics() gin.HandlerFunc {
	return func(c *gin.Context) {
		start := time.Now()
		c.Next()
		path := c.FullPath() // the ROUTE pattern ("/users/:id"), not the raw path —
		if path == "" {      // avoids unbounded label cardinality from IDs
			path = "unknown"
		}
		status := strconv.Itoa(c.Writer.Status())
		httpRequests.WithLabelValues(c.Request.Method, path, status).Inc()
		httpLatency.WithLabelValues(c.Request.Method, path).Observe(time.Since(start).Seconds())
	}
}
// r.GET("/metrics", gin.WrapH(promhttp.Handler()))
```

> **Gotcha:** Label by `c.FullPath()` (the route template), never the raw URL — using `/users/123`, `/users/124`, … as labels creates unbounded cardinality and will OOM Prometheus.

### 14.3 Tracing (OpenTelemetry, overview) **[A]**

Distributed tracing follows one request across services (API → DB → downstream API) as a tree of timed **spans**, so you can see *where* latency goes. Use **OpenTelemetry**: wrap the engine with `otelgin` middleware, propagate trace context via headers, and export spans to a collector (Jaeger/Tempo). Link the request ID and trace ID so a log line points to its trace.

```go
// import "go.opentelemetry.io/contrib/instrumentation/github.com/gin-gonic/gin/otelgin"
// r.Use(otelgin.Middleware("my-api"))
// Then create child spans around DB/external calls using the request's context.
```

### 14.4 Health & readiness endpoints **[A]**

Load balancers and orchestrators (Kubernetes) need to know if an instance should receive traffic. Two distinct checks:

- **Liveness** (`/healthz`): "is the process up?" — cheap, no dependencies. Failure ⇒ restart the pod.
- **Readiness** (`/readyz`): "can it serve requests *right now*?" — checks dependencies (DB ping). Failure ⇒ stop routing traffic here, but don't restart.

```go
r.GET("/healthz", func(c *gin.Context) { c.JSON(200, gin.H{"status": "ok"}) })

r.GET("/readyz", func(c *gin.Context) {
	ctx, cancel := context.WithTimeout(c.Request.Context(), 2*time.Second)
	defer cancel()
	if err := pool.Ping(ctx); err != nil {
		c.JSON(503, gin.H{"status": "not_ready", "db": "down"})
		return
	}
	c.JSON(200, gin.H{"status": "ready"})
})
```

> **Best practice:** Exclude `/healthz`, `/readyz`, and `/metrics` from auth and access logging (they're noisy). Keep them dependency-light so a slow dependency doesn't make liveness flap and trigger needless restarts.

---

## 15. Centralized Error Handling

Scattered, inconsistent error responses are a maintenance nightmare and a security risk (leaking internals). Centralize: define a domain error model, map it to HTTP in **one** place, and always return the same envelope.

### 15.1 Domain errors → HTTP status mapping **[A]**

The service returns *domain* errors; a single transport-layer mapper converts them to status codes + the §5.5 envelope. Add a typed error for cases that carry a status/code.

```go
package httpx

// A typed app error the service can return when it wants an explicit code.
type AppError struct {
	Status  int
	Code    string
	Message string
}

func (e *AppError) Error() string { return e.Message }

func NewAppError(status int, code, msg string) *AppError {
	return &AppError{Status: status, Code: code, Message: msg}
}

// RespondError is THE one place that turns any error into an HTTP response.
func RespondError(c *gin.Context, err error) {
	traceID := c.GetString("request_id")

	// 1. Explicit AppError → use its status/code.
	var appErr *AppError
	if errors.As(err, &appErr) {
		c.JSON(appErr.Status, ErrorResponse{Error: ErrorBody{
			Code: appErr.Code, Message: appErr.Message, TraceID: traceID}})
		return
	}

	// 2. Known domain sentinels → mapped statuses.
	switch {
	case errors.Is(err, user.ErrNotFound), errors.Is(err, order.ErrNotFound):
		c.JSON(404, ErrorResponse{Error: ErrorBody{Code: "not_found",
			Message: "resource not found", TraceID: traceID}})
	case errors.Is(err, user.ErrEmailTaken):
		c.JSON(409, ErrorResponse{Error: ErrorBody{Code: "conflict",
			Message: "email already registered", TraceID: traceID}})
	default:
		// 3. Unknown error → 500 with a GENERIC message (never leak err.Error()).
		//    Log the real error server-side, keyed by traceID, for debugging.
		slog.Error("unhandled error", slog.Any("err", err), slog.String("request_id", traceID))
		c.JSON(500, ErrorResponse{Error: ErrorBody{Code: "internal_error",
			Message: "internal server error", TraceID: traceID}})
	}
}
```

> **Security recommendation:** For unmapped/500 errors, return a generic message and a trace ID; log the detailed error internally. Never return raw `err.Error()`, stack traces, SQL errors, or driver messages to clients — they leak schema and implementation details that aid attackers.

### 15.2 Handler usage **[A]**

```go
func (h *Handler) GetByID(c *gin.Context) {
	u, err := h.svc.Get(c.Request.Context(), c.Param("id"))
	if err != nil {
		httpx.RespondError(c, err) // one call handles every error kind consistently
		return
	}
	c.JSON(200, toUserResponse(u))
}
```

Validation errors get the field-level treatment from §6.6 (422); panic recovery (§10.4) catches the truly unexpected. Together these three — validation 422, mapped domain errors, recovered 500 — cover every failure path with a consistent envelope.

---

## 16. Performance & Scale

A service "scales" when you handle more load by adding instances (horizontal scale) without rewrites. The enablers: statelessness, timeouts everywhere, pooling, caching, graceful lifecycle, and bounded concurrency.

### 16.1 Graceful shutdown **[A]**

On deploy, the orchestrator sends SIGTERM. You must stop accepting new requests but **finish in-flight ones** before exiting, or you'll drop user requests mid-flight. Run the server in a goroutine and block on a signal, then `Shutdown` with a timeout.

```go
func run(cfg *config.Config, handler http.Handler) error {
	srv := &http.Server{
		Addr:              cfg.Addr,
		Handler:           handler,
		ReadHeaderTimeout: 5 * time.Second,
		ReadTimeout:       15 * time.Second,
		WriteTimeout:      15 * time.Second,
		IdleTimeout:       60 * time.Second,
	}

	// Start serving in the background.
	go func() {
		if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
			slog.Error("listen", slog.Any("err", err))
		}
	}()
	slog.Info("server started", slog.String("addr", cfg.Addr))

	// Block until SIGINT/SIGTERM.
	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	<-quit
	slog.Info("shutting down...")

	// Give in-flight requests time to finish, then stop.
	ctx, cancel := context.WithTimeout(context.Background(), cfg.ShutdownTimeout)
	defer cancel()
	return srv.Shutdown(ctx) // stops accepting, waits for active requests up to the timeout
}
```

### 16.2 Timeouts everywhere **[A]**

Every boundary needs a deadline or a slow/hung peer ties up resources forever: server `ReadHeaderTimeout`/`ReadTimeout`/`WriteTimeout` (above), a per-request timeout middleware (§10.6), and `context`-bounded DB/external calls (§12). Without `ReadHeaderTimeout` you're exposed to Slowloris (§17).

### 16.3 Statelessness & horizontal scaling **[A]**

Keep **zero per-client state in process memory** (no in-RAM sessions, no local-only caches that must stay consistent). Then any instance serves any request, and you scale by running more identical instances behind a load balancer. Shared state (sessions, rate-limit counters, cache) goes in **Redis**; persistent state in the DB; uploaded files in object storage (§7.7), not local disk.

### 16.4 Caching with Redis **[A]**

Cache expensive, frequently-read, rarely-changing data to cut DB load and latency. The **cache-aside** pattern: look in cache; on miss, read DB, write cache with a TTL. See `REDIS_GUIDE.md` for depth.

```go
func (s *Service) GetCached(ctx context.Context, id string) (User, error) {
	key := "user:" + id
	if b, err := s.rdb.Get(ctx, key).Bytes(); err == nil { // hit
		var u User
		_ = json.Unmarshal(b, &u)
		return u, nil
	}
	u, err := s.repo.FindByID(ctx, id) // miss → source of truth
	if err != nil {
		return User{}, err
	}
	b, _ := json.Marshal(u)
	s.rdb.Set(ctx, key, b, 5*time.Minute) // populate with TTL
	return u, nil
}
// On update/delete, invalidate: s.rdb.Del(ctx, "user:"+id)
```

### 16.5 Goroutine safety & bounded concurrency **[A]**

Shared mutable state touched by multiple requests must be synchronized (`sync.Mutex`/`RWMutex`) — Gin handlers run concurrently. For fan-out work (calling several downstreams) bound concurrency with a worker pool or `errgroup`, so a spike doesn't spawn unbounded goroutines and exhaust memory/connections. Always run `go test -race` (§18).

### 16.6 Profiling **[A]**

When something is slow, profile — don't guess. `net/http/pprof` exposes CPU/heap/goroutine profiles; analyse with `go tool pprof`. Gate it behind admin auth or an internal-only port — never expose `/debug/pprof` publicly.

---

## 17. Security Hardening

Security is not one feature; it's a checklist applied everywhere. This consolidates the OWASP-aligned essentials for a Gin API. Most map to specific sections above.

### 17.1 The checklist **[A]**

- **Input validation (everywhere):** bind into DTOs with `binding` tags (§6); allowlist enums/sort fields with `oneof`; cap sizes and counts. Treat all input as hostile.
- **SQL injection:** parameterised queries only (§12.2); never concatenate input into SQL; allowlist dynamic column/sort names.
- **Authn/authz:** short-lived JWTs with pinned algorithm (§11.3); RBAC + service-layer ownership checks (§11.5); strong password hashing (§11.2).
- **Secrets:** from env/secret-manager, never committed or logged (§13.3).
- **Transport security (TLS):** terminate TLS at the proxy/ingress or in-process (`srv.ListenAndServeTLS`); redirect HTTP→HTTPS; set HSTS.
- **Secure response headers:** see §17.2.
- **CORS correctly:** explicit origin allowlist; no `*` with credentials (§10.5).
- **Rate limiting / brute-force:** global + stricter per-route limits on auth endpoints (§10.6); lockout/backoff on repeated login failures; `Retry-After`.
- **Request size limits:** `MaxBytesReader` + `gin-contrib/size` (§7.6) to prevent memory exhaustion.
- **File-upload security:** the §7.9 checklist (type by magic bytes, safe names, size caps, store outside webroot, safe `Content-Type`).
- **Slowloris / slow-read:** set `ReadHeaderTimeout`/`ReadTimeout` (§16.1).
- **Don't leak in errors/logs:** generic 500s (§15); never log tokens, passwords, full bodies, or PII (§10.3).
- **Dependency scanning:** run `govulncheck` in CI (§17.3).
- **Disable debug in prod:** `gin.ReleaseMode`; no `/debug/pprof` exposed; no verbose error bodies.

### 17.2 Secure headers middleware **[A]**

```go
func SecureHeaders() gin.HandlerFunc {
	return func(c *gin.Context) {
		h := c.Writer.Header()
		h.Set("X-Content-Type-Options", "nosniff")          // no MIME sniffing
		h.Set("X-Frame-Options", "DENY")                    // clickjacking defence
		h.Set("Referrer-Policy", "no-referrer")
		h.Set("Content-Security-Policy", "default-src 'none'; frame-ancestors 'none'") // tighten for JSON APIs
		if c.Request.TLS != nil {
			h.Set("Strict-Transport-Security", "max-age=63072000; includeSubDomains") // HSTS
		}
		c.Next()
	}
}
```

### 17.3 Dependency & vulnerability scanning **[A]**

```bash
go install golang.org/x/vuln/cmd/govulncheck@latest
govulncheck ./...   # flags known CVEs in your deps AND whether you actually call the vulnerable code
go list -m -u all   # see available dependency updates
```

> **Security recommendation:** Run `govulncheck ./...` and `go vet ./...` in CI on every PR; pin dependency versions; review `go.sum` changes. Keep the Go toolchain and Gin current — security fixes land in patch releases.

---

## 18. Testing

The layered architecture pays off most in tests: the service is tested with a fake repo (no DB, microseconds), handlers with `httptest` (no network), and a few integration tests exercise the real stack.

### 18.1 Unit-testing the service with a fake repository **[I/A]**

Because the service depends on the `Repository` *interface*, inject the in-memory fake (§9.3). No database, fast and deterministic. Use **table-driven tests** — Go's idiom for covering many cases compactly.

```go
package user_test

import (
	"context"
	"errors"
	"testing"

	"github.com/you/my-api/internal/user"
)

func TestService_Register(t *testing.T) {
	tests := []struct {
		name      string
		seedEmail string // pre-existing email in the repo, if any
		email     string
		wantErr   error
	}{
		{name: "success", email: "new@example.com", wantErr: nil},
		{name: "duplicate email", seedEmail: "dup@example.com", email: "dup@example.com", wantErr: user.ErrEmailTaken},
	}

	for _, tc := range tests {
		t.Run(tc.name, func(t *testing.T) {
			repo := user.NewMemoryRepo() // fake — no DB
			if tc.seedEmail != "" {
				_ = repo.Save(context.Background(), user.User{ID: "seed", Email: tc.seedEmail})
			}
			svc := user.NewService(repo, fakeHasher{})

			_, err := svc.Register(context.Background(), tc.email, "Alice", "password1234")

			if !errors.Is(err, tc.wantErr) {
				t.Fatalf("got err %v, want %v", err, tc.wantErr)
			}
		})
	}
}

// A trivial fake hasher keeps the test pure (no bcrypt cost).
type fakeHasher struct{}

func (fakeHasher) Hash(p string) (string, error)        { return "hashed:" + p, nil }
func (fakeHasher) Verify(p, e string) (bool, error)     { return e == "hashed:"+p, nil }
```

### 18.2 Mocking the service for handler tests **[A]**

The handler depends on a *service interface* (define one even if there's a single impl, just for testability). Pass a mock that returns canned results, and assert the HTTP status/body.

```go
// A small hand-written mock — no mocking framework needed for one method.
type mockUserSvc struct {
	getFn func(ctx context.Context, id string) (user.User, error)
}

func (m mockUserSvc) Get(ctx context.Context, id string) (user.User, error) {
	return m.getFn(ctx, id)
}
```

### 18.3 Handler tests with httptest + Gin test mode **[I/A]**

Use `gin.SetMode(gin.TestMode)`, build the router, and drive it with `httptest.NewRecorder()` + `httptest.NewRequest()` via `router.ServeHTTP` — no real socket.

```go
func TestUserHandler_GetByID(t *testing.T) {
	gin.SetMode(gin.TestMode)

	t.Run("found → 200", func(t *testing.T) {
		svc := mockUserSvc{getFn: func(_ context.Context, id string) (user.User, error) {
			return user.User{ID: id, Email: "a@b.com", Name: "Al"}, nil
		}}
		r := gin.New()
		h := user.NewHandler(svc)
		r.GET("/users/:id", h.GetByID)

		req := httptest.NewRequest(http.MethodGet, "/users/42", nil)
		w := httptest.NewRecorder()
		r.ServeHTTP(w, req)

		if w.Code != http.StatusOK {
			t.Fatalf("status = %d, want 200; body=%s", w.Code, w.Body.String())
		}
		var got user.UserResponse
		_ = json.Unmarshal(w.Body.Bytes(), &got)
		if got.ID != "42" {
			t.Errorf("id = %q, want 42", got.ID)
		}
	})

	t.Run("not found → 404", func(t *testing.T) {
		svc := mockUserSvc{getFn: func(_ context.Context, _ string) (user.User, error) {
			return user.User{}, user.ErrNotFound
		}}
		r := gin.New()
		r.GET("/users/:id", user.NewHandler(svc).GetByID)

		req := httptest.NewRequest(http.MethodGet, "/users/nope", nil)
		w := httptest.NewRecorder()
		r.ServeHTTP(w, req)

		if w.Code != http.StatusNotFound {
			t.Fatalf("status = %d, want 404", w.Code)
		}
	})
}
```

### 18.4 Testing a POST with a JSON body and validation **[I]**

```go
func TestCreateUser_ValidationError(t *testing.T) {
	gin.SetMode(gin.TestMode)
	r := buildTestRouter() // your router with the create route registered

	body, _ := json.Marshal(map[string]any{"email": "not-an-email"}) // missing name/password
	req := httptest.NewRequest(http.MethodPost, "/api/v1/users", bytes.NewReader(body))
	req.Header.Set("Content-Type", "application/json")
	w := httptest.NewRecorder()
	r.ServeHTTP(w, req)

	if w.Code != http.StatusUnprocessableEntity {
		t.Fatalf("status = %d, want 422", w.Code)
	}
}
```

### 18.5 Integration tests with testcontainers (note) **[A]**

Unit tests use fakes; a handful of **integration tests** should exercise the real pgx repository against a real Postgres to catch SQL/mapping bugs the fake can't. **`testcontainers-go`** spins up a throwaway Postgres in Docker for the test, runs migrations, and tears it down — real DB, no shared environment. Tag these tests (`//go:build integration`) so they run separately from fast unit tests.

```go
// //go:build integration
// pg, _ := postgres.RunContainer(ctx, ...) — start a real Postgres
// dsn := pg.MustConnectionString(ctx)
// runMigrations(dsn); repo := user.NewPostgresRepo(pool)
// ...assert real Save/FindByID round-trips...
```

```bash
go test ./...                      # fast unit tests
go test -race ./...                # with the race detector (always in CI)
go test -tags=integration ./...    # include integration tests
go test -cover ./...               # coverage
```

---

## 19. Deployment

Ship the service as a small, static container image, configured by environment, behind a reverse proxy, with health checks and graceful shutdown. See `DOCKER_GUIDE.md` for Docker depth.

### 19.1 Multi-stage Docker build (tiny static image) **[A]**

A **multi-stage** build compiles in a full Go image, then copies *only the binary* into a minimal final image — a few MB instead of hundreds, with a smaller attack surface.

```dockerfile
# ---- build stage ----
FROM golang:1.24 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download                       # cached layer: deps change rarely
COPY . .
# CGO off → a fully static binary; trim symbols for size; target linux.
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /app ./cmd/api

# ---- final stage ----
FROM gcr.io/distroless/static-debian12:nonroot   # no shell, no package manager, runs as nonroot
WORKDIR /
COPY --from=build /app /app
COPY migrations /migrations                       # if migrating at startup
EXPOSE 8080
USER nonroot:nonroot                              # never run as root
ENTRYPOINT ["/app"]
```

> **Best practice:** Use `distroless` or `scratch` (with CA certs added if you make outbound TLS calls) for the smallest, most secure image. Pin the base image by digest in production. Don't bake secrets or `.env` into the image — inject at runtime.

### 19.2 Behind a reverse proxy, health checks, 12-factor **[A]**

- **Reverse proxy / ingress** (Nginx, Caddy, a cloud LB, or Kubernetes Ingress) terminates TLS, load-balances across instances, and forwards to your container. Configure Gin's trusted proxies (§4.6) to match.
- **Health checks:** point the orchestrator's liveness probe at `/healthz` and readiness at `/readyz` (§14.4). Readiness gates traffic during startup/migrations.
- **Graceful shutdown** (§16.1) + a sufficient `terminationGracePeriodSeconds` so SIGTERM lets in-flight requests drain.
- **12-factor config** (§13): one image, many environments, configured purely by env vars.

### 19.3 CI/CD (note) **[A]**

A typical pipeline on each PR: `go vet` → `golangci-lint` → `go test -race -cover` → `govulncheck` → build image → push to registry → deploy (staging, then prod after approval). Run migrations as a separate, ordered step before rolling out new code that depends on them.

---

## 20. Complete Worked Example: Users + Orders API

This is a small but *real* slice of the architecture end-to-end, in the **feature-based** layout (§8.2): config → DB → repository → service → handler → middleware → router → `main` with graceful shutdown, plus a fake-repo unit test. Two features (`user`, `order`) and a `shared` package. Imports are shown; obvious repetition (full pgx scans for every method) is trimmed with comments to keep it readable.

```
my-api/
├── cmd/api/main.go
├── internal/
│   ├── shared/httpx/respond.go
│   ├── user/{user.go, dto.go, repository.go, service.go, handler.go, service_test.go}
│   └── order/{order.go, dto.go, repository.go, service.go, handler.go}
└── go.mod
```

### 20.1 Shared response/error helpers **[A]**

```go
// internal/shared/httpx/respond.go
package httpx

import (
	"errors"
	"log/slog"
	"net/http"

	"github.com/gin-gonic/gin"
)

type ErrorBody struct {
	Code    string `json:"code"`
	Message string `json:"message"`
	TraceID string `json:"trace_id,omitempty"`
}
type ErrorResponse struct {
	Error ErrorBody `json:"error"`
}

// AppError lets a service request a specific status/code.
type AppError struct {
	Status  int
	Code    string
	Message string
}

func (e *AppError) Error() string { return e.Message }

// RespondError maps any error to a consistent envelope. Feature packages register
// their sentinel→status mappings via RegisterMapping at init.
type mapping struct {
	is     error
	status int
	code   string
}

var mappings []mapping

func RegisterMapping(sentinel error, status int, code string) {
	mappings = append(mappings, mapping{sentinel, status, code})
}

func RespondError(c *gin.Context, err error) {
	trace := c.GetString("request_id")

	var appErr *AppError
	if errors.As(err, &appErr) {
		c.JSON(appErr.Status, ErrorResponse{ErrorBody{appErr.Code, appErr.Message, trace}})
		return
	}
	for _, m := range mappings {
		if errors.Is(err, m.is) {
			c.JSON(m.status, ErrorResponse{ErrorBody{m.code, m.is.Error(), trace}})
			return
		}
	}
	slog.Error("unhandled error", slog.Any("err", err), slog.String("request_id", trace))
	c.JSON(http.StatusInternalServerError,
		ErrorResponse{ErrorBody{"internal_error", "internal server error", trace}})
}
```

### 20.2 The user feature **[A]**

```go
// internal/user/user.go — domain + port
package user

import (
	"context"
	"errors"
	"time"
)

type User struct {
	ID           string
	Email        string
	Name         string
	PasswordHash string
	Role         string
	CreatedAt    time.Time
}

var (
	ErrNotFound   = errors.New("user not found")
	ErrEmailTaken = errors.New("email already registered")
)

type Repository interface {
	Save(ctx context.Context, u User) error
	FindByID(ctx context.Context, id string) (User, error)
	FindByEmail(ctx context.Context, email string) (User, error)
}

type PasswordHasher interface {
	Hash(plain string) (string, error)
	Verify(plain, encoded string) (bool, error)
}
```

```go
// internal/user/dto.go — wire shapes + mappers
package user

import "time"

type CreateRequest struct {
	Email    string `json:"email"    binding:"required,email"`
	Name     string `json:"name"     binding:"required,min=2,max=100"`
	Password string `json:"password" binding:"required,min=10,max=128"`
}

type Response struct {
	ID        string    `json:"id"`
	Email     string    `json:"email"`
	Name      string    `json:"name"`
	Role      string    `json:"role"`
	CreatedAt time.Time `json:"created_at"`
}

func toResponse(u User) Response {
	return Response{ID: u.ID, Email: u.Email, Name: u.Name, Role: u.Role, CreatedAt: u.CreatedAt}
}
```

```go
// internal/user/service.go — use-cases (no HTTP, no SQL)
package user

import (
	"context"
	"errors"
	"time"

	"github.com/google/uuid"
)

type Service struct {
	repo   Repository
	hasher PasswordHasher
}

func NewService(repo Repository, hasher PasswordHasher) *Service {
	return &Service{repo: repo, hasher: hasher}
}

func (s *Service) Register(ctx context.Context, email, name, password string) (User, error) {
	if _, err := s.repo.FindByEmail(ctx, email); err == nil {
		return User{}, ErrEmailTaken
	} else if !errors.Is(err, ErrNotFound) {
		return User{}, err
	}
	hash, err := s.hasher.Hash(password)
	if err != nil {
		return User{}, err
	}
	u := User{
		ID: uuid.NewString(), Email: email, Name: name,
		PasswordHash: hash, Role: "user", CreatedAt: time.Now().UTC(),
	}
	if err := s.repo.Save(ctx, u); err != nil {
		return User{}, err
	}
	return u, nil
}

func (s *Service) Get(ctx context.Context, id string) (User, error) {
	return s.repo.FindByID(ctx, id)
}
```

```go
// internal/user/repository.go — pgx adapter (+ register error mappings)
package user

import (
	"context"
	"errors"
	"net/http"

	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgxpool"

	"github.com/you/my-api/internal/shared/httpx"
)

func init() {
	httpx.RegisterMapping(ErrNotFound, http.StatusNotFound, "not_found")
	httpx.RegisterMapping(ErrEmailTaken, http.StatusConflict, "conflict")
}

type PostgresRepo struct{ pool *pgxpool.Pool }

func NewPostgresRepo(pool *pgxpool.Pool) *PostgresRepo { return &PostgresRepo{pool: pool} }

func (r *PostgresRepo) Save(ctx context.Context, u User) error {
	_, err := r.pool.Exec(ctx,
		`INSERT INTO users (id, email, name, password_hash, role, created_at)
		 VALUES ($1,$2,$3,$4,$5,$6)`,
		u.ID, u.Email, u.Name, u.PasswordHash, u.Role, u.CreatedAt)
	return err
}

func (r *PostgresRepo) FindByID(ctx context.Context, id string) (User, error) {
	var u User
	err := r.pool.QueryRow(ctx,
		`SELECT id,email,name,password_hash,role,created_at FROM users WHERE id=$1`, id).
		Scan(&u.ID, &u.Email, &u.Name, &u.PasswordHash, &u.Role, &u.CreatedAt)
	if errors.Is(err, pgx.ErrNoRows) {
		return User{}, ErrNotFound
	}
	return u, err
}

func (r *PostgresRepo) FindByEmail(ctx context.Context, email string) (User, error) {
	var u User
	err := r.pool.QueryRow(ctx,
		`SELECT id,email,name,password_hash,role,created_at FROM users WHERE email=$1`, email).
		Scan(&u.ID, &u.Email, &u.Name, &u.PasswordHash, &u.Role, &u.CreatedAt)
	if errors.Is(err, pgx.ErrNoRows) {
		return User{}, ErrNotFound
	}
	return u, err
}
```

```go
// internal/user/repository_memory.go — fake for tests
package user

import (
	"context"
	"sync"
)

type MemoryRepo struct {
	mu   sync.RWMutex
	byID map[string]User
}

func NewMemoryRepo() *MemoryRepo { return &MemoryRepo{byID: map[string]User{}} }

func (r *MemoryRepo) Save(ctx context.Context, u User) error {
	r.mu.Lock()
	defer r.mu.Unlock()
	for _, e := range r.byID {
		if e.Email == u.Email && e.ID != u.ID {
			return ErrEmailTaken
		}
	}
	r.byID[u.ID] = u
	return nil
}
func (r *MemoryRepo) FindByID(ctx context.Context, id string) (User, error) {
	r.mu.RLock()
	defer r.mu.RUnlock()
	if u, ok := r.byID[id]; ok {
		return u, nil
	}
	return User{}, ErrNotFound
}
func (r *MemoryRepo) FindByEmail(ctx context.Context, email string) (User, error) {
	r.mu.RLock()
	defer r.mu.RUnlock()
	for _, u := range r.byID {
		if u.Email == email {
			return u, nil
		}
	}
	return User{}, ErrNotFound
}
```

```go
// internal/user/handler.go — transport (the only place that touches gin)
package user

import (
	"context"
	"net/http"

	"github.com/gin-gonic/gin"

	"github.com/you/my-api/internal/shared/httpx"
)

// UseCases is the interface the handler needs — defined here for testability.
type UseCases interface {
	Register(ctx context.Context, email, name, password string) (User, error)
	Get(ctx context.Context, id string) (User, error)
}

type Handler struct{ svc UseCases }

func NewHandler(svc UseCases) *Handler { return &Handler{svc: svc} }

func (h *Handler) RegisterRoutes(rg *gin.RouterGroup) {
	rg.POST("/users", h.Create)
	rg.GET("/users/:id", h.GetByID)
}

func (h *Handler) Create(c *gin.Context) {
	var req CreateRequest
	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(http.StatusUnprocessableEntity, httpx.ErrorResponse{
			Error: httpx.ErrorBody{Code: "validation_failed", Message: err.Error(),
				TraceID: c.GetString("request_id")}})
		return
	}
	u, err := h.svc.Register(c.Request.Context(), req.Email, req.Name, req.Password)
	if err != nil {
		httpx.RespondError(c, err)
		return
	}
	c.JSON(http.StatusCreated, toResponse(u))
}

func (h *Handler) GetByID(c *gin.Context) {
	u, err := h.svc.Get(c.Request.Context(), c.Param("id"))
	if err != nil {
		httpx.RespondError(c, err)
		return
	}
	c.JSON(http.StatusOK, toResponse(u))
}
```

### 20.3 The order feature (abbreviated) **[A]**

```go
// internal/order/order.go
package order

import (
	"context"
	"errors"
	"time"
)

type Order struct {
	ID        string
	UserID    string
	Total     int64 // cents — never float for money
	Status    string
	CreatedAt time.Time
}

var ErrNotFound = errors.New("order not found")

type Repository interface {
	Save(ctx context.Context, o Order) error
	ListByUser(ctx context.Context, userID string, limit int) ([]Order, error)
}
```

```go
// internal/order/service.go — note dependency on the USER service via a small port,
// not on the user package's internals (low coupling between features).
package order

import (
	"context"
	"time"

	"github.com/google/uuid"
)

// UserChecker is the slice of "user" that order actually needs. Defined here
// (consumer side) so order depends on a tiny interface, not the whole user pkg.
type UserChecker interface {
	Get(ctx context.Context, id string) (struct{ ID string }, error)
}

type Service struct{ repo Repository }

func NewService(repo Repository) *Service { return &Service{repo: repo} }

func (s *Service) Place(ctx context.Context, userID string, total int64) (Order, error) {
	o := Order{
		ID: uuid.NewString(), UserID: userID, Total: total,
		Status: "pending", CreatedAt: time.Now().UTC(),
	}
	if err := s.repo.Save(ctx, o); err != nil {
		return Order{}, err
	}
	return o, nil
}

func (s *Service) ListForUser(ctx context.Context, userID string, limit int) ([]Order, error) {
	if limit <= 0 || limit > 100 {
		limit = 20
	}
	return s.repo.ListByUser(ctx, userID, limit)
}
```

### 20.4 Composition root — `main.go` with graceful shutdown **[A]**

```go
// cmd/api/main.go
package main

import (
	"context"
	"errors"
	"log/slog"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/gin-gonic/gin"
	"golang.org/x/crypto/bcrypt"

	"github.com/you/my-api/internal/config"
	"github.com/you/my-api/internal/order"
	"github.com/you/my-api/internal/shared/db"
	"github.com/you/my-api/internal/shared/middleware"
	"github.com/you/my-api/internal/user"
)

// bcryptHasher satisfies user.PasswordHasher.
type bcryptHasher struct{}

func (bcryptHasher) Hash(p string) (string, error) {
	b, err := bcrypt.GenerateFromPassword([]byte(p), 12)
	return string(b), err
}
func (bcryptHasher) Verify(p, e string) (bool, error) {
	return bcrypt.CompareHashAndPassword([]byte(e), []byte(p)) == nil, nil
}

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
	slog.SetDefault(logger)

	// 1. Config — fail fast.
	cfg, err := config.Load()
	if err != nil {
		logger.Error("config", slog.Any("err", err))
		os.Exit(1)
	}
	gin.SetMode(cfg.GinMode)

	// 2. Infrastructure.
	ctx := context.Background()
	pool, err := db.NewPool(ctx, cfg.DatabaseURL)
	if err != nil {
		logger.Error("db", slog.Any("err", err))
		os.Exit(1)
	}
	defer pool.Close()

	// 3. Wire features (composition root: the ONLY place naming concrete types).
	userSvc := user.NewService(user.NewPostgresRepo(pool), bcryptHasher{})
	userHandler := user.NewHandler(userSvc)

	orderSvc := order.NewService(order.NewPostgresRepo(pool))
	orderHandler := order.NewHandler(orderSvc)

	// 4. Build the router + middleware.
	r := gin.New()
	r.Use(middleware.Recovery(logger), middleware.RequestID(), middleware.Logger(logger))

	r.GET("/healthz", func(c *gin.Context) { c.JSON(200, gin.H{"status": "ok"}) })
	r.GET("/readyz", func(c *gin.Context) {
		cx, cancel := context.WithTimeout(c.Request.Context(), 2*time.Second)
		defer cancel()
		if err := pool.Ping(cx); err != nil {
			c.JSON(503, gin.H{"status": "not_ready"})
			return
		}
		c.JSON(200, gin.H{"status": "ready"})
	})

	v1 := r.Group("/api/v1")
	userHandler.RegisterRoutes(v1)
	orderHandler.RegisterRoutes(v1)

	// 5. Serve with graceful shutdown.
	srv := &http.Server{
		Addr:              cfg.Addr,
		Handler:           r,
		ReadHeaderTimeout: 5 * time.Second,
		ReadTimeout:       15 * time.Second,
		WriteTimeout:      15 * time.Second,
		IdleTimeout:       60 * time.Second,
	}
	go func() {
		if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
			logger.Error("listen", slog.Any("err", err))
		}
	}()
	logger.Info("listening", slog.String("addr", cfg.Addr))

	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	<-quit
	logger.Info("shutting down")

	shCtx, cancel := context.WithTimeout(context.Background(), cfg.ShutdownTimeout)
	defer cancel()
	if err := srv.Shutdown(shCtx); err != nil {
		logger.Error("shutdown", slog.Any("err", err))
	}
}
```

### 20.5 The fake-repo unit test **[A]**

```go
// internal/user/service_test.go
package user_test

import (
	"context"
	"errors"
	"testing"

	"github.com/you/my-api/internal/user"
)

type fakeHasher struct{}

func (fakeHasher) Hash(p string) (string, error)    { return "h:" + p, nil }
func (fakeHasher) Verify(p, e string) (bool, error) { return e == "h:"+p, nil }

func TestService_Register(t *testing.T) {
	tests := []struct {
		name    string
		seed    string
		email   string
		wantErr error
	}{
		{"success", "", "new@example.com", nil},
		{"duplicate", "dup@example.com", "dup@example.com", user.ErrEmailTaken},
	}
	for _, tc := range tests {
		t.Run(tc.name, func(t *testing.T) {
			repo := user.NewMemoryRepo()
			if tc.seed != "" {
				_ = repo.Save(context.Background(), user.User{ID: "seed", Email: tc.seed})
			}
			svc := user.NewService(repo, fakeHasher{})

			u, err := svc.Register(context.Background(), tc.email, "Alice", "password1234")
			if !errors.Is(err, tc.wantErr) {
				t.Fatalf("err = %v, want %v", err, tc.wantErr)
			}
			if tc.wantErr == nil {
				if u.ID == "" || u.Role != "user" || u.PasswordHash != "h:password1234" {
					t.Errorf("unexpected user: %+v", u)
				}
			}
		})
	}
}
```

This example exercises every layer: typed config validated at startup, a pooled DB, repositories behind interfaces (pgx for prod, memory for tests), services holding the business rules, handlers that only translate HTTP, cross-cutting middleware, a versioned route group, graceful shutdown, and a fast unit test with zero infrastructure.

---

## 21. Gotchas & Best Practices

### 21.1 Context & lifecycle

- **Never use `*gin.Context` after the handler returns or in a goroutine** — it's pooled and recycled. Use `c.Copy()` for a snapshot and `c.Request.Context()` for cancellation (§4.1).
- **Pass `c.Request.Context()` to every DB/external call** so client disconnects and timeouts cancel work instead of leaking connections.

### 21.2 Binding & validation

- **Prefer `ShouldBind*` over `Bind*`** so you control error format/status (§6.1).
- **The body is read once.** Re-reading after binding yields nothing; use `ShouldBindBodyWith` if both middleware and handler need it (§4.2).
- **`required` treats zero as missing.** Use `*int`/`*bool` pointers with `omitempty` when 0/false are valid (§6.5).
- **Query binding uses the `form` tag, not `json`.**
- **Allowlist sort/filter fields with `oneof`** and map to real columns — never interpolate into SQL (§5.4).

### 21.3 Responses & chain control

- **`c.Abort()` does not `return`** — you must `return` after it, or the handler keeps running.
- **Render once.** A second `c.JSON` (or `c.JSON` after an abort-with-JSON) causes a `superfluous WriteHeader` warning and a corrupt response.
- **`c.AbortWithStatusJSON` already wrote the response** — don't write again.

### 21.4 Routing & engine

- **Set `gin.SetMode` before creating the engine.** Drive it from config; never ship debug mode.
- **Configure trusted proxies** (`SetTrustedProxies`) or `ClientIP()` is spoofable (§4.6).
- **Use `gin.New()` + explicit middleware in production**, but never forget `Recovery` (§2.2).
- **Don't expose `/debug/pprof`, raw errors, or stack traces** to clients.

### 21.5 Architecture

- **Keep `gin` out of service/domain layers** — only the transport layer imports it, so the core is portable and unit-testable.
- **Define interfaces where they're consumed** (the service), not next to implementations — keeps dependencies pointing inward.
- **`main` is the only place that names concrete types** (the composition root).
- **Don't add the repository/interface layers prematurely** for a trivial app — add them when you have a testing need or a second implementation.
- **Use cents (int64) for money, never float.**

### 21.6 Concurrency & resources

- **Gin handlers run concurrently** — guard shared mutable state with a mutex, and run `go test -race`.
- **Bound fan-out concurrency** (worker pool / `errgroup`) so spikes don't exhaust memory/connections.
- **Always set server timeouts** and implement graceful shutdown (§16).

### 21.7 Security recap

- Parameterised SQL; validate all input; safe file uploads (§7.9); explicit CORS; secure headers; rate-limit auth; short-lived JWTs with pinned alg; secrets from a manager; generic 500s; `govulncheck` in CI (§17).

### 21.8 Quick reference — production middleware order

| # | Middleware | Why this position |
|---|---|---|
| 1 | Recovery | Outermost — must catch panics from everything below |
| 2 | RequestID | So even failures are logged with a correlation ID |
| 3 | Logger | Times the whole chain; logs on the way out |
| 4 | SecureHeaders | Cheap, applies to all responses |
| 5 | CORS | Before routing decisions/preflight |
| 6 | RateLimit | Reject early, before expensive work |
| 7 | Auth (group only) | Only on protected routes |
| 8 | Handler | Your code |

---

## 22. Study Path & Build-to-Learn Projects

Work through these in order; each project forces the relevant concepts to stick. Read the cross-referenced guides alongside.

### Phase 1 — Gin foundations (Week 1)

Read §1–§4. **Project: Hello API** — `GET /ping`, `GET /users/:id` (hardcoded), `POST /echo` (echo the JSON body). Goals: engine, routing, params/query, `*gin.Context`, `c.JSON`.

### Phase 2 — REST + binding/validation (Week 2)

Read §5–§6. **Project: Products CRUD** — full CRUD with DTOs, validator tags, pagination/filtering/sorting, the consistent success/error envelopes, proper status codes (201/204/404/422). In-memory store behind a `Repository` interface. Goals: REST design, binding, validation, error formatting.

### Phase 3 — File uploads (Week 3)

Read §7. **Project: Image Upload Service** — single + multiple uploads; validate size, magic-byte type, real-image decode + dimension caps; UUID filenames; serve via `r.Static`; return URLs. Then add streaming for large files and a pre-signed-URL path to object storage. Goals: secure upload handling end-to-end.

### Phase 4 — Architecture, middleware, auth (Week 4)

Read §8–§11 and `GO_LANG_AND_PATTERNS_GUIDE.md` + `JWT_AUTH_ARGON2_GUIDE.md`. **Project: refactor Phase 2/3 into the feature-based structure** — handler→service→repository, DI in `main`, custom middleware (request-ID, structured logging, recovery, CORS, rate limit), JWT access/refresh auth, RBAC, secure password hashing. Goals: clean architecture, the two folder structures, middleware chains, authn/authz.

### Phase 5 — Database & observability (Week 5)

Read §12, §14 and `POSTGRESQL_GUIDE.md` + `RELATIONAL_DB_DESIGN_GUIDE.md` + `REDIS_GUIDE.md`. **Project: wire a real Postgres** via pgx + pgxpool, golang-migrate migrations, transactions across two repositories, fix an N+1, add Redis cache-aside, add `/healthz` + `/readyz`, structured logs, Prometheus `/metrics`. Goals: real persistence, transactions, caching, observability.

### Phase 6 — Hardening, testing, deployment (Week 6)

Read §13, §15–§19 and `DOCKER_GUIDE.md`. **Project: production-ready the service** — typed config validated at startup; centralized error mapping; security headers + `govulncheck`; unit tests (fake repo + httptest), one testcontainers integration test, `go test -race`; multi-stage distroless Dockerfile; graceful shutdown; CI pipeline. Goals: ship something you'd be comfortable putting real users on.

### Capstone — the full §20 example, extended

Take the Users + Orders API and add: refresh-token rotation with revocation, order state machine, idempotency keys on order creation, pagination on orders, OpenTelemetry tracing, and a load test (e.g. `k6`/`vegeta`) to find and profile a bottleneck with `pprof`.

### Reference bookmarks (offline-friendly)

| Topic | Location |
|---|---|
| Gin API | `pkg.go.dev/github.com/gin-gonic/gin` |
| Validator tags | `pkg.go.dev/github.com/go-playground/validator/v10` |
| pgx / pgxpool | `pkg.go.dev/github.com/jackc/pgx/v5` |
| golang-jwt | `pkg.go.dev/github.com/golang-jwt/jwt/v5` |
| golang-migrate | `pkg.go.dev/github.com/golang-migrate/migrate/v4` |
| slog | `pkg.go.dev/log/slog` |
| httptest | `pkg.go.dev/net/http/httptest` |
| Companion guides | `GO_LANG_AND_PATTERNS_GUIDE.md`, `JWT_AUTH_ARGON2_GUIDE.md`, `POSTGRESQL_GUIDE.md`, `RELATIONAL_DB_DESIGN_GUIDE.md`, `REDIS_GUIDE.md`, `DOCKER_GUIDE.md`, `GO_FILESYSTEM_OS_CLI_GUIDE.md` |

---

*Guide accurate as of June 2026 — Gin v1.10+, Go 1.25/1.26. Confirm version-sensitive details against `pkg.go.dev` when online.*
