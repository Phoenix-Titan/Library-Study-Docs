# Go net/http — REST API — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Anyone going from "I can write a `func main()`" to "I can design, build, secure, test, and operate a production REST API using **only the Go standard library**" — entirely offline. No Gin, Echo, Chi, or Fiber required. Every concept is explained in **prose first** — what it is, the *logic/why*, what it's for and when to reach for it, how to use it, the key parameters, best practices, and **security recommendations** — and only *then* shown as heavily-commented, runnable code you can paste into a `main.go` and run with `go run .`. Read top-to-bottom the first time; afterwards use the Table of Contents as a lookup. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **Go 1.23 / 1.24** (current in 2026). The single most important feature for API authors is the **enhanced `net/http.ServeMux` shipped in Go 1.22** (February 2024): the standard router now supports **method-based routing**, **wildcard path segments** (`{id}`), and **catch-all tails** (`{path...}`) natively — features that previously forced everyone onto a third-party router. Other modern stdlib features used throughout: `log/slog` structured logging (1.21), `min`/`max`/`clear` builtins (1.21), `signal.NotifyContext` (1.16), and `//go:embed` (1.16). Where behaviour is version-sensitive it is flagged with **⚡ Version note**. The author is on **Windows 11**, so cross-platform notes are called out. Always confirm exact API signatures at pkg.go.dev/net/http.
>
> **This guide's place in the library.** `net/http` is the **standard-library foundation** that every Go web framework is built on top of. Read this *first*. The companion **`GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md`** shows how Gin packages these exact ideas (handlers, middleware, routing, binding) into a higher-level ergonomic API — once you understand the primitives here, Gin's conveniences are obvious rather than magical. For the **language itself** (slices, interfaces, goroutines, channels, generics, errors) see **`GO_GUIDE.md`** and **`GO_LANG_AND_PATTERNS_GUIDE.md`**; this guide assumes you can read Go and focuses on the HTTP/REST domain. Cross-references to those guides appear inline where a language concept (interfaces, `context`, closures) is load-bearing.

---

## Table of Contents

1. [Why `net/http` — the standard library is the framework](#1-why-nethttp--the-standard-library-is-the-framework) **[B]**
2. [Hello server & the foundational types (Handler, HandlerFunc, ResponseWriter, Request)](#2-hello-server--the-foundational-types) **[B]**
3. [ServeMux & routing — classic vs the Go 1.22+ enhanced router](#3-servemux--routing) **[B/I]**
4. [Reading requests in depth — body, JSON, params, headers, forms, uploads](#4-reading-requests-in-depth) **[B/I]**
5. [Writing responses in depth — status, headers, JSON, errors](#5-writing-responses-in-depth) **[B/I]**
6. [REST resource design — modeling your API](#6-rest-resource-design) **[I]**
7. [The middleware pattern — wrappers, decorators, composition](#7-the-middleware-pattern) **[I]**
8. [Full CRUD REST API — in-memory then interface-based store](#8-full-crud-rest-api) **[I]**
9. [Error handling & JSON error responses](#9-error-handling--json-error-responses) **[I]**
10. [Validation](#10-validation) **[I]**
11. [Context — cancellation, deadlines, values](#11-context--cancellation-deadlines-values) **[I/A]**
12. [Structured logging with `log/slog`](#12-structured-logging-with-logslog) **[I]**
13. [Graceful shutdown & signal handling](#13-graceful-shutdown--signal-handling) **[I/A]**
14. [Security hardening — timeouts, body limits, headers, TLS, input](#14-security-hardening) **[I/A]**
15. [`http.Client` — making outbound requests](#15-httpclient--making-outbound-requests) **[I]**
16. [Serving static files & embedding](#16-serving-static-files--embedding) **[I]**
17. [Testing handlers with `net/http/httptest`](#17-testing-handlers-with-nethttphttptest) **[I/A]**
18. [Structuring a real service — project layout](#18-structuring-a-real-service) **[I/A]**
19. [Tips, tricks & gotchas](#19-tips-tricks--gotchas) **[A]**
20. [Study path & build-to-learn projects](#20-study-path--build-to-learn-projects)

---

## 1. Why `net/http` — the standard library is the framework

### 1.1 The standard library is production-grade **[B]**

Newcomers from Node (Express), Python (Flask/FastAPI), or Ruby (Rails) instinctively look for "the Go web framework" to install. The important truth is **you usually don't need one.** The standard library's `net/http` ships a *fully concurrent, production-ready* HTTP/1.1 and HTTP/2 server and client. Concretely it gives you:

- A **multiplexer (router)** — `http.ServeMux` — that since Go 1.22 supports method-based and wildcard-path routing (§3).
- **TLS** via `ListenAndServeTLS`, with **HTTP/2 negotiated automatically** over TLS.
- **Keep-alive** connection management and connection pooling (server and client).
- **Streaming** request and response bodies (you are not forced to buffer whole payloads).
- A powerful, pooled **`http.Client`** for outbound calls (§15).
- **Cookies, headers, multipart/form-data**, gzip on the client, range requests on the file server.
- A built-in **file server** (§16) and a first-class **testing** package, `net/http/httptest` (§17).

**The logic / why this matters.** Every Go web framework — Gin, Echo, Chi, Fiber's `net/http` mode — is ultimately a thin (or not-so-thin) layer over `http.Handler`. The concepts are identical: a request comes in, a handler runs, a response goes out, and middleware wraps handlers. If you learn the primitives here, *every* framework becomes "the same thing with nicer syntax." If you skip them, frameworks feel like magic and you can't debug the layer beneath you.

**When to reach for a framework anyway.** Frameworks add real convenience: terse routing groups, automatic request binding + validation, a built-in middleware ecosystem, and (in Gin's case) a fast radix-tree router with helpers like `c.JSON`. Reach for one when you value that ergonomics over zero dependencies, or when your team already standardizes on it. But even large APIs run perfectly well on `net/http`. The honest recommendation: **start on `net/http`, understand it, and adopt a framework only when you can name the specific limitation it solves for you.** (See `GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md` for the Gin path — it maps one-to-one onto the ideas here.)

### 1.2 What lives in the package — a map **[B]**

You don't need to memorize this, but knowing the cast of characters makes the rest of the guide read faster. Each symbol is explained in depth in its section.

| Symbol | Role |
|---|---|
| `http.ListenAndServe(addr, handler)` | Start an HTTP server (convenience; no timeouts — avoid in prod) |
| `http.ListenAndServeTLS(addr, cert, key, h)` | Start an HTTPS server |
| `http.Server` | The **configurable** server struct (timeouts, TLS, handler) — use this in prod |
| `http.ServeMux` | Request multiplexer / router |
| `http.Handler` | The core **interface**: anything with `ServeHTTP(ResponseWriter, *Request)` |
| `http.HandlerFunc` | Adapter converting a plain function into an `http.Handler` |
| `http.ResponseWriter` | Interface for writing the response (headers, status, body) |
| `*http.Request` | The incoming request (method, URL, headers, body, context) |
| `http.Client` / `http.DefaultClient` | Outbound HTTP client |
| `http.FileServer` / `http.StripPrefix` | Serve files from disk / strip a URL prefix |
| `http.Error` / `http.NotFound` / `http.Redirect` | Convenience response helpers |
| `http.MaxBytesReader` | Cap request body size (security — §14) |
| `http.Cookie` / `http.SetCookie` | Cookie support |

### 1.3 The request lifecycle — the mental model **[B]**

Before any code, hold this picture in your head. It is the whole game:

1. A client opens a TCP connection and sends an HTTP request (method, path, headers, optional body).
2. `http.Server` accepts the connection and reads the request, building a `*http.Request`. **Each request is served in its own goroutine** — concurrency is automatic, which is *why* shared state needs locks (§8) and *why* a panic in one handler must be recovered so it doesn't take down the connection (§7).
3. The server calls your top-level handler's `ServeHTTP(w, r)`. If that handler is a `ServeMux`, it inspects the method+path and dispatches to the matching registered handler.
4. Middleware (if any) wraps the chain, running code before and after the inner handler (§7).
5. Your handler reads from `r`, does work, and writes to `w` (status code, headers, body).
6. When the handler returns, the server finishes the response and may keep the connection alive for reuse.

Everything else in this guide is a detail of one of those six steps.

---

## 2. Hello server & the foundational types

### 2.1 The minimal server **[B]**

**What it is.** The smallest useful program: register one function to handle requests and start listening. **The logic:** `http.HandleFunc` records "for this path, call this function"; `http.ListenAndServe` binds a TCP port and blocks forever, serving requests as they arrive. **How to use it / parameters:** the first arg to `ListenAndServe` is the address (`":8080"` means "all interfaces, port 8080"; `"127.0.0.1:8080"` restricts to localhost — a small security win in dev); the second is the handler (`nil` means "use the package-global default mux," which we'll move away from shortly).

```go
// main.go
package main

import (
	"fmt"
	"log"
	"net/http"
)

// A handler function has this exact signature. w is where we write the
// response; r holds everything about the incoming request.
func helloHandler(w http.ResponseWriter, r *http.Request) {
	// Fprintln writes to the ResponseWriter. The first Write implicitly
	// sends a "200 OK" status (see §2.4).
	fmt.Fprintln(w, "Hello, World!")
}

func main() {
	// Register helloHandler for "/". With the enhanced mux, "/" is a
	// subtree pattern that matches every path not matched by something
	// more specific.
	http.HandleFunc("/", helloHandler)

	// ListenAndServe blocks until the server errors. log.Fatal prints the
	// error and exits non-zero. In production prefer an explicit
	// http.Server with timeouts (§14) — this convenience form has none.
	log.Println("Listening on http://localhost:8080")
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

Run and test:

```bash
go run .
# in another terminal:
curl http://localhost:8080/
# -> Hello, World!
```

> **Best practice from the very first program:** prefer `127.0.0.1:8080` in local dev so your laptop isn't exposing a server to the whole LAN, and graduate to an explicit `http.Server{}` (§14) the moment you write anything real.

### 2.2 The `http.Handler` interface — the keystone **[B]**

**What it is.** A single, tiny interface that *everything* in `net/http` is built around:

```go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```

**The logic / why this is the most important paragraph in the guide.** "A handler is any value with a `ServeHTTP` method." That's it. Because the router, every endpoint, the file server, and every piece of middleware all satisfy this *same* interface, they compose freely: a `ServeMux` is a handler that contains handlers; a middleware is a handler that wraps a handler; the server takes one top-level handler and calls its `ServeHTTP`. This is interface-driven design (see `GO_GUIDE.md` §9 on methods & interfaces) doing its job — uniformity through one small contract. If you internalize "it's handlers all the way down," the rest of `net/http` stops having special cases.

**How to use it.** You can implement `ServeHTTP` on your own struct when a handler needs to carry dependencies *and* you want it to be a handler directly:

```go
// A handler can be a struct — useful when it needs dependencies (a DB,
// a logger, config). It becomes an http.Handler by defining ServeHTTP.
type GreetHandler struct {
	greeting string // a dependency injected at construction time
}

func (h GreetHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "%s from a struct handler!\n", h.greeting)
}

func main() {
	mux := http.NewServeMux()
	// mux.Handle takes anything implementing http.Handler.
	mux.Handle("/greet", GreetHandler{greeting: "Hello"})
	log.Fatal(http.ListenAndServe(":8080", mux))
}
```

### 2.3 `http.HandlerFunc` — the function adapter **[B]**

**What it is.** Writing a struct + `ServeHTTP` for every trivial endpoint is verbose. `http.HandlerFunc` is an **adapter type**: a named function type that *itself* satisfies `http.Handler` by calling the underlying function. **The logic** is a classic Go trick — you can attach methods to a function type, so a function can "become" an interface value:

```go
// This is literally how the standard library defines it:
//
//   type HandlerFunc func(ResponseWriter, *Request)
//
//   func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
//       f(w, r)   // calling ServeHTTP just calls the function
//   }
//
// So a plain function, converted to HandlerFunc, is an http.Handler.

mux := http.NewServeMux()

// These two registrations are exactly equivalent:

// Explicit: wrap the function in HandlerFunc, register with Handle.
mux.Handle("/ping", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintln(w, "pong")
}))

// Shortcut: HandleFunc does the HandlerFunc wrapping for you.
mux.HandleFunc("/ping2", func(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintln(w, "pong")
})
```

**When to use which.** Use `HandleFunc` (plain function) for most endpoints. Use a struct + `ServeHTTP` (or, more commonly, *methods* on a struct used with `HandleFunc` — see §8) when the handler needs injected dependencies.

### 2.4 `http.ResponseWriter` — writing the response **[B]**

**What it is.** An interface with exactly three methods, representing the response you're building:

```go
type ResponseWriter interface {
	// Header returns the response header map. You MUST set headers BEFORE
	// the first Write or WriteHeader, because those flush the headers to
	// the wire. Setting a header afterwards is silently ignored.
	Header() Header

	// Write writes body bytes. If WriteHeader has not been called yet, the
	// first Write implicitly calls WriteHeader(http.StatusOK) — i.e. 200.
	Write([]byte) (int, error)

	// WriteHeader sends the status line + headers. It can only take effect
	// ONCE; later calls are ignored (and log a warning).
	WriteHeader(statusCode int)
}
```

**The logic / why the ordering rule exists.** HTTP sends the status line and headers *before* the body, in that order, on the wire. So once you write any body byte (or explicitly call `WriteHeader`), the headers are committed and frozen. This is the single most common beginner bug (§19): setting a header after writing. The mental rule is **headers → status → body, always in that order.**

```go
func myHandler(w http.ResponseWriter, r *http.Request) {
	// 1. Headers first.
	w.Header().Set("Content-Type", "application/json")
	w.Header().Set("X-Request-Id", "abc123")

	// 2. Then the status code.
	w.WriteHeader(http.StatusCreated) // 201

	// 3. Then the body.
	w.Write([]byte(`{"id":1}`))
}
```

**Type assertions for extra power.** `ResponseWriter` is *just* the three methods, but the concrete value the server passes you often implements more — `http.Flusher` (push buffered bytes immediately, for streaming/SSE), `http.Hijacker` (take over the raw TCP connection, for WebSockets). You access these via type assertions (`if f, ok := w.(http.Flusher); ok { f.Flush() }`) — shown in §11.

### 2.5 `*http.Request` — the key fields **[B]**

**What it is.** A struct holding everything about the incoming request. You'll touch a handful of fields constantly:

```go
func tour(w http.ResponseWriter, r *http.Request) {
	_ = r.Method       // "GET", "POST", "PUT", "PATCH", "DELETE", ...
	_ = r.URL          // *url.URL: r.URL.Path, r.URL.RawQuery, r.URL.Query()
	_ = r.Header       // http.Header (map[string][]string) — request headers
	_ = r.Body         // io.ReadCloser — the request body; always close it
	_ = r.Context()    // context.Context — cancellation, deadlines, values (§11)
	_ = r.RemoteAddr   // "IP:port" of the client (but see §14 re: proxies)
	_ = r.TLS          // non-nil only if the connection is HTTPS
	_ = r.Host         // the Host header / authority
	// After r.ParseForm():
	_ = r.Form         // url.Values: query + body form fields
	_ = r.PostForm     // url.Values: body form fields only
	// Go 1.22+ routing:
	_ = r.PathValue("id") // value of an {id} wildcard segment (§3)
}
```

**Best practice / security note up front:** treat *every* field of `r` as **untrusted input**. The body, headers, query string, and even `r.RemoteAddr` (when behind a proxy) are attacker-controlled. Validate sizes (§14), validate values (§10), and never interpolate request data into SQL, shell commands, file paths, or HTML without proper escaping/parameterization.

---

## 3. ServeMux & routing

### 3.1 What a multiplexer is, and the classic ServeMux **[B]**

**What it is.** A "mux" (multiplexer) is a **router**: it inspects the incoming request and decides *which handler* should serve it. `http.ServeMux` is the standard-library router and is itself an `http.Handler` (so the server just calls its `ServeHTTP`, and it dispatches internally).

**The logic / why it was historically weak.** Before Go 1.22, `ServeMux` was deliberately minimal — it matched only fixed strings and "subtree" prefixes, with **no method routing and no path parameters.** That gap is the entire reason third-party routers (chi, gorilla/mux, httprouter) existed and became near-mandatory. Knowing the old behaviour still matters because (a) you'll read old code and (b) the trailing-slash subtree rule survives into the new mux.

```go
mux := http.NewServeMux()

// Exact match: ONLY the path "/about".
mux.HandleFunc("/about", aboutHandler)

// Subtree match (trailing slash): "/static/" matches "/static/",
// "/static/css/app.css", "/static/anything/at/all". The trailing slash
// is what makes it a prefix match rather than an exact match.
mux.HandleFunc("/static/", staticHandler)
```

### 3.2 The Go 1.22+ enhanced ServeMux **[B/I]**

⚡ **Version note:** Go 1.22 (Feb 2024) shipped a dramatically improved `ServeMux`. You now get **method routing**, **named wildcard segments**, and **catch-all tails** natively — the features that previously sent everyone to a third-party router. This is *the* reason "use a router library" is no longer the default advice.

**What changed — the new pattern syntax.** A registration string can now be `"METHOD /path/with/{wildcards}"`. The method (if present) restricts matching to that HTTP verb; `{name}` captures one path segment; `{name...}` captures everything to the end of the path.

#### Method-based patterns **[B]**

**Why:** restricting a route to a verb is the heart of REST — `GET /users` and `POST /users` are *different operations on the same resource*. Pre-1.22 you had to write `if r.Method != ...` inside every handler; now the router does it, and it automatically returns **405 Method Not Allowed** (with a correct `Allow` header) when a path matches but the method doesn't.

```go
mux := http.NewServeMux()

// Pattern form: "METHOD /path"
mux.HandleFunc("GET /users", listUsersHandler)    // only GET
mux.HandleFunc("POST /users", createUserHandler)  // only POST

// Method + wildcard segment — the CRUD-by-id shape:
mux.HandleFunc("GET /users/{id}", getUserHandler)
mux.HandleFunc("PUT /users/{id}", updateUserHandler)
mux.HandleFunc("DELETE /users/{id}", deleteUserHandler)
```

#### Wildcard segments `{name}` **[B]**

**What:** `{id}` matches exactly **one** path segment (no slashes). **How to read the value:** `r.PathValue("id")`. It always returns a `string` (the URL is text), so parse it yourself.

```go
// {id} matches one segment, e.g. "42" in GET /books/42 but NOT "42/reviews".
mux.HandleFunc("GET /books/{id}", func(w http.ResponseWriter, r *http.Request) {
	id := r.PathValue("id") // "42"
	fmt.Fprintf(w, "Book ID: %s\n", id)
})
```

#### Catch-all (tail) wildcard `{name...}` **[I]**

**What:** `{path...}` (note the trailing `...`) matches **the entire rest of the path including slashes.** **When:** file servers, proxy-style handlers, anything where the remainder is itself a path.

```go
// {path...} captures everything after /files/, slashes included.
mux.HandleFunc("GET /files/{path...}", func(w http.ResponseWriter, r *http.Request) {
	rest := r.PathValue("path") // "images/logo.png" for /files/images/logo.png
	fmt.Fprintf(w, "Serving: %s\n", rest)
})
```

#### Exact-match anchor `{$}` **[I]**

⚡ **Version note (1.22+):** the special wildcard `{$}` anchors a pattern to match *only* that exact path, defeating subtree matching. `GET /{$}` matches the root `/` and nothing else, so a request to `/nope` falls through to your 404 handler instead of being swallowed by `/`.

```go
mux.HandleFunc("GET /{$}", homeHandler) // matches only "/", not "/anything"
```

### 3.3 Routing precedence — how conflicts resolve **[I]**

**The logic / why this matters.** When two patterns could match the same request, you need a deterministic winner. Go 1.22+ resolves by **specificity, not registration order** — the more specific pattern wins. And critically, if two patterns are *equally* specific and overlap, the mux **panics at startup**. That panic is a feature: it forces you to fix genuinely ambiguous routes instead of shipping a coin-flip.

| Priority | Beats less-specific when the pattern is… | Example |
|---|---|---|
| 1 (highest) | Method + exact path | `GET /users/me` |
| 2 | Method + wildcard path | `GET /users/{id}` |
| 3 | No method + exact path | `/users/me` |
| 4 | No method + wildcard path | `/users/{id}` |
| 5 (lowest) | Subtree (trailing slash) | `/` |

```go
// FINE: "GET /users/me" is strictly more specific than "GET /users/{id}",
// so /users/me hits getMe and /users/42 hits getUser. No ambiguity.
mux.HandleFunc("GET /users/me", getMeHandler)
mux.HandleFunc("GET /users/{id}", getUserHandler)
```

> **Gotcha:** registering two overlapping equally-specific patterns (e.g. `GET /a/{x}/c` and `GET /a/b/{y}`) panics on startup. Restructure so one is clearly more specific, or use distinct prefixes.

### 3.4 A full routing example **[B/I]**

```go
package main

import (
	"fmt"
	"log"
	"net/http"
)

func main() {
	mux := http.NewServeMux()

	// No method prefix => matches ANY method on this exact path.
	mux.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, `{"status":"ok"}`)
	})

	// Users resource — the canonical REST verb set.
	mux.HandleFunc("GET /users", listUsers)
	mux.HandleFunc("POST /users", createUser)
	mux.HandleFunc("GET /users/{id}", getUser)
	mux.HandleFunc("PUT /users/{id}", updateUser)
	mux.HandleFunc("DELETE /users/{id}", deleteUser)

	// Nested resource: posts belonging to a user. Multiple wildcards are fine.
	mux.HandleFunc("GET /users/{userID}/posts/{postID}", getPost)

	log.Fatal(http.ListenAndServe(":8080", mux))
}

func listUsers(w http.ResponseWriter, r *http.Request)  { fmt.Fprintln(w, "list users") }
func createUser(w http.ResponseWriter, r *http.Request) { fmt.Fprintln(w, "create user") }
func getUser(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "get user %s\n", r.PathValue("id"))
}
func updateUser(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "update user %s\n", r.PathValue("id"))
}
func deleteUser(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "delete user %s\n", r.PathValue("id"))
}
func getPost(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "user=%s post=%s\n", r.PathValue("userID"), r.PathValue("postID"))
}
```

### 3.5 Your own mux vs the default mux **[B]**

**The logic / why you must not use the default.** `http.HandleFunc`/`http.Handle` (package-level) register on a hidden global `http.DefaultServeMux`. Global mutable state is bad here for two reasons: (1) it makes tests interfere with each other, and (2) **any imported package that calls `http.HandleFunc` in its `init()` silently adds routes to your server** — including, historically, `net/http/pprof`, which would expose debug endpoints you didn't ask for. **Always create an explicit `http.NewServeMux()`.**

```go
// AVOID in real code: package-global default mux.
http.HandleFunc("/", handler)
http.ListenAndServe(":8080", nil) // nil => DefaultServeMux

// PREFER: an explicit, local, testable mux with no global state.
mux := http.NewServeMux()
mux.HandleFunc("/", handler)
http.ListenAndServe(":8080", mux)
```

### 3.6 Route groups & sub-routers **[I]**

`net/http` has no built-in "router group" like Gin's `r.Group("/api/v1")`, but `ServeMux` composes: mount one mux inside another with `StripPrefix`. **When:** versioning (`/api/v1`), or applying middleware to a subtree.

```go
// Build a sub-router for the v1 API.
v1 := http.NewServeMux()
v1.HandleFunc("GET /users", listUsers) // path here is relative to the mount

root := http.NewServeMux()
// Mount v1 under /api/v1/. StripPrefix removes "/api/v1" so v1 sees "/users".
// The trailing slash makes "/api/v1/" a subtree the inner mux handles.
root.Handle("/api/v1/", http.StripPrefix("/api/v1", v1))
```

---

## 4. Reading requests in depth

### 4.1 Path parameters with `r.PathValue` **[B]**

⚡ **Version note:** `r.PathValue` (and `r.SetPathValue`, used in tests) is **Go 1.22+**. On older Go you need a third-party router. **What it does:** returns the captured value of a `{name}` segment as a string; returns `""` if the current route has no such wildcard (so validate — don't assume non-empty).

```go
mux.HandleFunc("GET /products/{category}/{id}", func(w http.ResponseWriter, r *http.Request) {
	category := r.PathValue("category") // "electronics"
	idStr := r.PathValue("id")          // "99" (always a string)

	// Parse + validate. NEVER trust that it's a valid integer.
	id, err := strconv.Atoi(idStr)
	if err != nil || id <= 0 {
		http.Error(w, "id must be a positive integer", http.StatusBadRequest)
		return
	}
	fmt.Fprintf(w, "category=%s id=%d\n", category, id)
})
```

### 4.2 Query parameters **[B]**

**What:** the `?key=value&...` part of the URL. **How:** `r.URL.Query()` parses it into `url.Values` (a `map[string][]string`) — it never errors. `.Get(key)` returns the first value (or `""`); index the map directly for repeated keys. **When:** filtering, pagination, search — anything optional/non-identifying. (Identifying data belongs in the path.)

```go
// URL: /search?q=golang&page=2&limit=20&tag=go&tag=web
mux.HandleFunc("GET /search", func(w http.ResponseWriter, r *http.Request) {
	q := r.URL.Query()
	query := q.Get("q")    // "golang" — "" if absent
	page := q.Get("page")  // "2"
	tags := q["tag"]       // []string{"go", "web"} — repeated key

	// Parse with sane defaults — query params are optional and untrusted.
	limit := 20
	if v := q.Get("limit"); v != "" {
		if n, err := strconv.Atoi(v); err == nil && n > 0 && n <= 100 {
			limit = n // clamp to a max to prevent abuse (security)
		}
	}
	fmt.Fprintf(w, "q=%s page=%s limit=%d tags=%v\n", query, page, limit, tags)
})
```

> **Security note:** always **clamp** numeric limits (page size, offset). An unbounded `limit` lets a client request a million rows and exhaust your memory/DB — a cheap denial-of-service.

### 4.3 Request headers **[B]**

**What:** request metadata as `http.Header` (a `map[string][]string`). **Key behaviour:** header names are **canonicalized** (`r.Header.Get("content-type")` and `Get("Content-Type")` are the same) so always use `.Get`/`.Set` rather than indexing the map directly with arbitrary casing.

```go
func headersDemo(w http.ResponseWriter, r *http.Request) {
	auth := r.Header.Get("Authorization") // "Bearer eyJ..."
	ct := r.Header.Get("Content-Type")    // "application/json"
	accept := r.Header.Get("Accept")
	encodings := r.Header["Accept-Encoding"] // []string for repeated headers
	_ = auth
	_ = ct
	_ = accept
	_ = encodings
}
```

### 4.4 Reading the body & decoding JSON — the careful way **[B/I]**

**What it is.** For REST APIs the request body is usually JSON. **The logic of decoding choices:**

- `json.NewDecoder(r.Body).Decode(&v)` **streams** from the body — it doesn't buffer the whole payload first, so it's more memory-efficient than `io.ReadAll` + `json.Unmarshal`. Prefer it.
- **Always cap the body** with `http.MaxBytesReader` *before* decoding (security — an attacker can otherwise stream gigabytes; §14).
- `dec.DisallowUnknownFields()` rejects payloads with unexpected fields — good for catching client typos and tightening your contract.
- **Always `defer r.Body.Close()`** so the connection can be reused.

**Key parameters / best practices, distilled:** cap size, disallow unknown fields, close the body, decode into a *request DTO* (a struct specific to this endpoint, not your domain model — see §6), then validate (§10) before touching storage.

```go
package main

import (
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"net/http"
)

type CreateUserRequest struct {
	Name  string `json:"name"`
	Email string `json:"email"`
	Age   int    `json:"age"`
}

func createUserHandler(w http.ResponseWriter, r *http.Request) {
	// SECURITY: cap the body at 1 MiB. MaxBytesReader also sends a clean
	// 413 if exceeded and stops the connection from streaming forever.
	r.Body = http.MaxBytesReader(w, r.Body, 1<<20)
	defer r.Body.Close()

	var req CreateUserRequest
	dec := json.NewDecoder(r.Body)
	dec.DisallowUnknownFields() // reject {"naem": ...} typos / unexpected fields

	if err := dec.Decode(&req); err != nil {
		// Distinguish "too big" from "malformed" for a better error message.
		var mbe *http.MaxBytesError
		if errors.As(err, &mbe) {
			http.Error(w, "request body too large", http.StatusRequestEntityTooLarge)
			return
		}
		http.Error(w, "invalid JSON: "+err.Error(), http.StatusBadRequest)
		return
	}

	// Reject a body with MORE than one JSON value (e.g. {...}{...}).
	if err := dec.Decode(&struct{}{}); err != io.EOF {
		http.Error(w, "body must contain a single JSON object", http.StatusBadRequest)
		return
	}

	// Validation comes next (§10) — never trust decoded data.
	if req.Name == "" {
		http.Error(w, "name is required", http.StatusUnprocessableEntity)
		return
	}
	fmt.Fprintf(w, "created user: %+v\n", req)
}
```

> ⚡ **Version note (Go 1.22+):** `*http.MaxBytesError` is the typed error returned when `MaxBytesReader`'s limit is exceeded; use `errors.As` to detect it. Older code string-matched the message — don't.

### 4.5 Form data (URL-encoded HTML forms) **[I]**

**When:** classic HTML `<form>` POSTs (`application/x-www-form-urlencoded`), or login forms. **How:** `r.ParseForm()` populates `r.Form` (query + body) and `r.PostForm` (body only). `r.FormValue(k)` is a shortcut that calls `ParseForm` for you.

```go
func formHandler(w http.ResponseWriter, r *http.Request) {
	if err := r.ParseForm(); err != nil {
		http.Error(w, "bad form", http.StatusBadRequest)
		return
	}
	name := r.FormValue("name")         // query OR body
	email := r.PostFormValue("email")   // body only — ignores ?email= in the URL
	fmt.Fprintf(w, "name=%s email=%s\n", name, email)
}
```

### 4.6 Multipart / file uploads **[I]**

**When:** file uploads, `multipart/form-data`. **Key parameter:** `ParseMultipartForm(maxMemory)` keeps up to `maxMemory` bytes in RAM and spills the rest to temp files on disk. **Security:** validate the content type and size of uploads; never trust `header.Filename` as a path (an attacker can send `../../etc/passwd` — sanitize with `filepath.Base`). For the full upload treatment (validation, streaming to disk, image checks) see `GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md`.

```go
func uploadHandler(w http.ResponseWriter, r *http.Request) {
	// 32 MiB kept in memory; the rest spills to temp files.
	if err := r.ParseMultipartForm(32 << 20); err != nil {
		http.Error(w, "bad multipart form", http.StatusBadRequest)
		return
	}
	file, header, err := r.FormFile("avatar")
	if err != nil {
		http.Error(w, "missing file field 'avatar'", http.StatusBadRequest)
		return
	}
	defer file.Close()

	// SECURITY: never use header.Filename directly as a path on disk.
	safeName := filepath.Base(header.Filename) // strips any directory components
	fmt.Fprintf(w, "uploaded %s (%d bytes)\n", safeName, header.Size)
	// Stream to disk with io.Copy(dstFile, file) after validating size/type.
}
```

---

## 5. Writing responses in depth

### 5.1 Status codes — use the constants **[B]**

**Why constants, not magic numbers:** `http.StatusUnprocessableEntity` is self-documenting; `422` is a guessing game. Knowing *which* code to return is part of API design — clients (and tooling) rely on the status to decide behaviour.

| Constant | Code | When to use |
|---|---|---|
| `StatusOK` | 200 | Successful GET/PUT/PATCH with a body |
| `StatusCreated` | 201 | Successful POST that created a resource (set `Location`) |
| `StatusNoContent` | 204 | Success with **no** body (DELETE, some PUTs) |
| `StatusBadRequest` | 400 | Malformed request (bad JSON, bad param type) |
| `StatusUnauthorized` | 401 | Missing/invalid authentication |
| `StatusForbidden` | 403 | Authenticated but not allowed |
| `StatusNotFound` | 404 | Resource doesn't exist |
| `StatusMethodNotAllowed` | 405 | Path exists, verb doesn't (mux sets this for you) |
| `StatusConflict` | 409 | State conflict (duplicate, version mismatch) |
| `StatusUnprocessableEntity` | 422 | Syntactically valid but **semantically** invalid (validation) |
| `StatusTooManyRequests` | 429 | Rate limited (set `Retry-After`) |
| `StatusInternalServerError` | 500 | Unexpected server error (never leak details) |

> **Design note on 400 vs 422:** use **400** when the request can't even be parsed (broken JSON, non-numeric id); use **422** when it parsed fine but fails business rules ("email already taken", "year must be > 0"). This split lets clients distinguish "fix your syntax" from "fix your data."

### 5.2 Setting headers, status, and body in order **[B]**

```go
func handler(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json") // 1. headers
	w.Header().Set("X-Request-Id", "abc123")
	w.WriteHeader(http.StatusCreated)                  // 2. status
	w.Write([]byte(`{"id":1,"name":"Alice"}`))          // 3. body
}
```

### 5.3 A reusable JSON encoding helper **[I]**

**The logic / why a helper.** You'll write JSON responses dozens of times; a single helper guarantees the `Content-Type` is always set, the status is always sent, and encoding is consistent. **Key subtlety:** once you've called `WriteHeader`, the headers are committed — if `Encode` then fails mid-stream you *cannot* change the status to 500, so the only correct move is to log. A more robust variant encodes to a buffer *first* so a serialization error can still produce a clean 500.

```go
package response // a tiny shared package, or just functions in main

import (
	"encoding/json"
	"log/slog"
	"net/http"
)

// WriteJSON encodes v as JSON with the given status. The buffer-first
// approach means a marshaling error becomes a clean 500 instead of a
// half-written body with a 200 already on the wire.
func WriteJSON(w http.ResponseWriter, status int, v any) {
	buf, err := json.Marshal(v)
	if err != nil {
		// Encoding failed BEFORE we sent anything — safe to send 500.
		http.Error(w, `{"error":"internal server error"}`, http.StatusInternalServerError)
		return
	}
	w.Header().Set("Content-Type", "application/json; charset=utf-8")
	w.WriteHeader(status)
	if _, err := w.Write(buf); err != nil {
		// Headers already sent; the client likely disconnected. Just log.
		slog.Error("write response", "err", err)
	}
}

// WriteError is a thin convenience over WriteJSON for the error shape.
func WriteError(w http.ResponseWriter, status int, message string) {
	WriteJSON(w, status, map[string]string{"error": message})
}
```

### 5.4 A consistent error response type **[I]**

Returning errors in a *predictable shape* (not free-form strings) lets clients parse them. See §9 for the full error-handling architecture; this is the wire format.

```go
// ErrorResponse is the single shape every error endpoint returns.
type ErrorResponse struct {
	Error  string            `json:"error"`            // human-readable summary
	Code   string            `json:"code,omitempty"`   // stable machine code, e.g. "USER_NOT_FOUND"
	Fields map[string]string `json:"fields,omitempty"` // per-field validation messages (§10)
}
```

---

## 6. REST resource design

**What REST is, briefly.** REST models your system as **resources** (nouns) addressed by **URLs**, manipulated with the standard HTTP **verbs**. The design discipline is: pick the nouns, map the verbs, and let HTTP status codes carry the outcome. Good resource design makes an API predictable — a client who knows one resource can guess the others.

**The conventions (memorize this table — it's the spine of every CRUD API):**

| Verb + path | Meaning | Success status | Body |
|---|---|---|---|
| `GET /books` | List the collection | 200 | array (often wrapped: `{data, count}`) |
| `POST /books` | Create one | 201 + `Location` header | the created resource |
| `GET /books/{id}` | Read one | 200 | the resource |
| `PUT /books/{id}` | Replace one (full) | 200 | the updated resource |
| `PATCH /books/{id}` | Update one (partial) | 200 | the updated resource |
| `DELETE /books/{id}` | Delete one | 204 | (none) |

**Best practices.**

- **Nouns, plural, lowercase:** `/books`, not `/getBooks` or `/Book`. The verb is the HTTP method.
- **Nest only one level deep:** `/users/{id}/posts` is fine; `/users/{id}/posts/{pid}/comments/{cid}/likes` is a smell — prefer `/comments/{cid}/likes`.
- **Separate DTOs from your domain model.** Define `CreateBookRequest`/`UpdateBookRequest` structs for *input* and a `Book` struct for *output/storage*. **The logic:** input and output have different fields (clients don't send `id` or `created_at`; you never accept those), and decoupling them stops a client from setting fields they shouldn't (mass-assignment vulnerability). This is a recurring REST security pattern.
- **Idempotency:** `GET`, `PUT`, `DELETE` should be idempotent (calling twice = same effect as once); `POST` typically is not. Design handlers accordingly.

```go
// OUTPUT / storage model — what the server owns.
type Book struct {
	ID        int       `json:"id"`
	Title     string    `json:"title"`
	Author    string    `json:"author"`
	Year      int       `json:"year"`
	CreatedAt time.Time `json:"created_at"`
}

// INPUT DTO for POST — note: no ID, no CreatedAt. The client cannot set them.
type CreateBookRequest struct {
	Title  string `json:"title"`
	Author string `json:"author"`
	Year   int    `json:"year"`
}

// INPUT DTO for PATCH — pointer fields distinguish "absent" from "zero".
// A nil *string means "client didn't send this field, leave it alone";
// a non-nil pointer to "" means "client explicitly set it to empty".
type UpdateBookRequest struct {
	Title  *string `json:"title"`
	Author *string `json:"author"`
	Year   *int    `json:"year"`
}
```

> **Why pointer fields in the PATCH DTO?** With plain `string`/`int`, you can't tell "the client omitted `year`" from "the client sent `year: 0`." Pointers (or a `map[string]any`) recover that distinction, which is exactly what partial updates need.

---

## 7. The middleware pattern

### 7.1 What middleware is and the logic of the wrapper pattern **[I]**

**What it is.** Middleware is a function that **wraps an `http.Handler` to add behaviour around it** — logging, panic recovery, CORS, auth, rate limiting, request IDs. The universal signature is:

```go
func(next http.Handler) http.Handler
```

**The logic / why this exact shape composes so cleanly.** Recall §2.2: *everything is an `http.Handler`.* A middleware takes a handler and returns a *new* handler that runs some code, calls `next.ServeHTTP`, then maybe runs more code. Because the result is itself a handler, you can feed it into another middleware, and another — they nest like Russian dolls. This is the **decorator pattern**, and it works precisely *because* of the single small interface. The wrapped call site looks like `A(B(C(handler)))`: a request flows **outermost-in** (A runs first, then B, then C, then the handler) and the response flows **innermost-out** (handler returns, then C's after-code, then B's, then A's).

**Key design rule:** a middleware either calls `next.ServeHTTP(w, r)` to continue the chain, **or** writes a response and `return`s to *stop* it (e.g. auth rejecting a request). Forgetting to `return` after writing an error — then falling through to `next` — is the classic middleware bug.

### 7.2 Logging middleware (and capturing the status code) **[I]**

**The subtlety:** to log the response *status*, you must capture it, but `ResponseWriter` gives no way to read back what status was written. The fix is a tiny wrapper type that records the status as it passes through — a recurring, important trick.

```go
package middleware

import (
	"log/slog"
	"net/http"
	"time"
)

// statusRecorder wraps ResponseWriter to remember the status code and
// bytes written, so logging/metrics middleware can report them.
type statusRecorder struct {
	http.ResponseWriter
	status int
	bytes  int
}

// WriteHeader intercepts the status before delegating to the real writer.
func (sr *statusRecorder) WriteHeader(code int) {
	sr.status = code
	sr.ResponseWriter.WriteHeader(code)
}

// Write records byte count and, if WriteHeader was never called, defaults
// the recorded status to 200 (matching net/http's implicit behaviour).
func (sr *statusRecorder) Write(b []byte) (int, error) {
	if sr.status == 0 {
		sr.status = http.StatusOK
	}
	n, err := sr.ResponseWriter.Write(b)
	sr.bytes += n
	return n, err
}

// Logger logs method, path, status, duration, and size for each request.
func Logger(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		rec := &statusRecorder{ResponseWriter: w, status: 0}

		next.ServeHTTP(rec, r) // run the rest of the chain

		slog.Info("request",
			"method", r.Method,
			"path", r.URL.Path,
			"status", rec.status,
			"bytes", rec.bytes,
			"duration", time.Since(start).String(),
			"remote", r.RemoteAddr,
		)
	})
}
```

### 7.3 Panic recovery middleware **[I]**

**Why you need it.** Each request runs in its own goroutine. An unrecovered panic in a handler crashes that goroutine and abruptly drops the client's connection (no clean 500). `Recoverer` turns panics into proper 500 responses and keeps the server healthy. **Best practice:** make it the **outermost** middleware so it catches panics from every layer below, and **never leak the panic detail to the client** — log it, return a generic message.

```go
package middleware

import (
	"log/slog"
	"net/http"
	"runtime/debug"
)

func Recoverer(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		defer func() {
			if rec := recover(); rec != nil {
				// Log the panic + stack for YOU; send a generic message to the client.
				slog.Error("panic recovered",
					"err", rec,
					"path", r.URL.Path,
					"stack", string(debug.Stack()),
				)
				// Only safe to set a status if nothing was written yet; in
				// practice handlers panic before writing. http.Error is fine here.
				http.Error(w, `{"error":"internal server error"}`, http.StatusInternalServerError)
			}
		}()
		next.ServeHTTP(w, r)
	})
}
```

### 7.4 CORS middleware **[I]**

**What CORS is.** Cross-Origin Resource Sharing is the browser security mechanism that controls whether a page on origin A may call your API on origin B. Your server signals permission via `Access-Control-*` headers. **The logic of preflight:** for "non-simple" requests (custom headers, `PUT`/`DELETE`, JSON content-type), the browser first sends an `OPTIONS` "preflight" asking permission; you must answer it (no body, 204) with the allowed methods/headers, *then* the browser sends the real request.

> **SECURITY — the #1 CORS mistake:** **never** combine `Access-Control-Allow-Origin: *` with `Access-Control-Allow-Credentials: true`. The spec forbids it, and doing so (if a browser allowed it) would let any site read authenticated responses. For credentialed APIs, **echo back a specific allowed origin** from an allow-list, never `*`.

```go
package middleware

import "net/http"

// CORS allows requests only from origins in the allow-list. Pass the exact
// origins you trust (scheme + host + port). Avoid "*" for credentialed APIs.
func CORS(allowed ...string) func(http.Handler) http.Handler {
	allow := make(map[string]bool, len(allowed))
	for _, o := range allowed {
		allow[o] = true
	}
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			origin := r.Header.Get("Origin")
			if origin != "" && allow[origin] {
				// Echo the specific origin (NOT "*") so credentials are allowed.
				w.Header().Set("Access-Control-Allow-Origin", origin)
				w.Header().Set("Vary", "Origin") // caches must vary by origin
				w.Header().Set("Access-Control-Allow-Credentials", "true")
				w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, PATCH, DELETE, OPTIONS")
				w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")
				w.Header().Set("Access-Control-Max-Age", "600") // cache preflight 10 min
			}
			// Answer preflight and STOP — no body, no downstream handler.
			if r.Method == http.MethodOptions {
				w.WriteHeader(http.StatusNoContent)
				return
			}
			next.ServeHTTP(w, r)
		})
	}
}
```

### 7.5 Auth middleware (Bearer token) **[I]**

**What:** gate routes behind authentication. The example uses a static Bearer token for clarity; in production validate a JWT (see `GO_GUIDE.md` for the language pieces, and a dedicated JWT/Argon2 guide for token/password specifics) and **stash the authenticated identity in the request context** (§11) so downstream handlers can read it.

> **SECURITY:** compare secrets with `subtle.ConstantTimeCompare`, not `==`. A normal `==` short-circuits on the first differing byte, leaking timing information an attacker can use to recover the token byte-by-byte.

```go
package middleware

import (
	"crypto/subtle"
	"net/http"
	"strings"
)

func RequireAuth(validToken string) func(http.Handler) http.Handler {
	want := []byte(validToken)
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			h := r.Header.Get("Authorization")
			token, ok := strings.CutPrefix(h, "Bearer ") // Go 1.20+ helper
			// Constant-time compare prevents timing attacks.
			if !ok || subtle.ConstantTimeCompare([]byte(token), want) != 1 {
				w.Header().Set("WWW-Authenticate", `Bearer`)
				http.Error(w, `{"error":"unauthorized"}`, http.StatusUnauthorized)
				return // STOP the chain — do not call next
			}
			next.ServeHTTP(w, r) // authenticated; continue
		})
	}
}
```

### 7.6 Rate-limiting middleware **[I/A]**

**Why:** protect against abuse and accidental floods. **How:** the stdlib `golang.org/x/time/rate` token-bucket limiter is the idiomatic tool (it's an official `x/` module, effectively standard). **The logic of a token bucket:** tokens refill at a steady rate up to a burst cap; each request spends one; when empty, reject with **429**. Below is a simple per-IP limiter; for production add eviction of stale IPs and consider a distributed store (Redis) across multiple instances.

```go
package middleware

import (
	"net"
	"net/http"
	"sync"

	"golang.org/x/time/rate"
)

// IPRateLimiter keeps one token bucket per client IP.
type IPRateLimiter struct {
	mu       sync.Mutex
	buckets  map[string]*rate.Limiter
	r        rate.Limit // tokens per second
	burst    int        // bucket capacity
}

func NewIPRateLimiter(perSec rate.Limit, burst int) *IPRateLimiter {
	return &IPRateLimiter{buckets: map[string]*rate.Limiter{}, r: perSec, burst: burst}
}

func (l *IPRateLimiter) limiter(ip string) *rate.Limiter {
	l.mu.Lock()
	defer l.mu.Unlock()
	lim, ok := l.buckets[ip]
	if !ok {
		lim = rate.NewLimiter(l.r, l.burst)
		l.buckets[ip] = lim
	}
	return lim
}

func (l *IPRateLimiter) Middleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// SECURITY: r.RemoteAddr is the direct peer. Behind a trusted proxy
		// you'd parse X-Forwarded-For, but ONLY if you trust the proxy.
		ip, _, _ := net.SplitHostPort(r.RemoteAddr)
		if !l.limiter(ip).Allow() {
			w.Header().Set("Retry-After", "1")
			http.Error(w, `{"error":"rate limit exceeded"}`, http.StatusTooManyRequests)
			return
		}
		next.ServeHTTP(w, r)
	})
}
```

### 7.7 Chaining middleware **[I]**

**The logic of order.** `Logger(Recoverer(CORS(mux)))` means: request enters Logger first, then Recoverer, then CORS, then the mux. So put **Recoverer outermost-ish** (to catch everything) and resource-specific middleware (auth) innermost. A tiny `Chain` helper makes the order readable left-to-right.

```go
// Chain applies middleware so the FIRST argument is the OUTERMOST layer.
// Chain(mux, Logger, Recoverer, CORS) => Logger(Recoverer(CORS(mux)))
func Chain(h http.Handler, mw ...func(http.Handler) http.Handler) http.Handler {
	for i := len(mw) - 1; i >= 0; i-- { // apply in reverse to preserve order
		h = mw[i](h)
	}
	return h
}

// Usage in main:
//   handler := middleware.Chain(mux,
//       middleware.Recoverer,                 // catches panics from all below
//       middleware.Logger,                    // logs every request
//       middleware.CORS("https://app.example.com"),
//   )
```

### 7.8 Per-route middleware **[I]**

Apply middleware to a single route by wrapping just that handler — perfect for protecting `/admin/*` while leaving public routes open.

```go
// Wrap one handler with auth; other routes stay public.
mux.Handle("GET /admin/users", middleware.RequireAuth(token)(http.HandlerFunc(adminListUsers)))
```

---

## 8. Full CRUD REST API

This section builds a complete, runnable **Books API** with all CRUD endpoints, a handler struct for dependency injection, an in-memory store, and — crucially — an **interface-based store** so you can swap in a database later without touching handlers. (For an *actual* SQL/ORM backend, see `GO_ENT_ORM_GUIDE.md`; the interface here is the seam those guides plug into.)

### 8.1 Domain model and DTOs **[I]**

(See §6 for why DTOs are separate from the model.)

```go
package main

import (
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"log"
	"net/http"
	"strconv"
	"sync"
	"time"
)

type Book struct {
	ID        int       `json:"id"`
	Title     string    `json:"title"`
	Author    string    `json:"author"`
	Year      int       `json:"year"`
	CreatedAt time.Time `json:"created_at"`
}

type CreateBookRequest struct {
	Title  string `json:"title"`
	Author string `json:"author"`
	Year   int    `json:"year"`
}

type UpdateBookRequest struct {
	Title  *string `json:"title"`  // pointers => "field omitted" vs "set to zero"
	Author *string `json:"author"`
	Year   *int    `json:"year"`
}
```

### 8.2 The store interface — designing for swappability **[I]**

**The logic / why an interface.** If your handlers depend on a *concrete* `*Store`, swapping the in-memory map for Postgres later means rewriting handlers. If they depend on an **interface**, you write the in-memory version today, the SQL version tomorrow, and a **mock** for tests — all interchangeable. This is dependency inversion, and it's the single most valuable structural decision in a Go service. Define the interface in terms of *what handlers need*, return a sentinel `ErrNotFound` (§9) so handlers can map it to a 404, and **thread `context.Context`** through every method (§11) so DB calls are cancellable.

```go
var ErrNotFound = errors.New("not found")

// BookStore is the contract handlers depend on. The in-memory store, a
// future SQL store, and test mocks all satisfy it.
type BookStore interface {
	List(ctx context.Context) ([]Book, error)
	Get(ctx context.Context, id int) (Book, error)
	Create(ctx context.Context, req CreateBookRequest) (Book, error)
	Update(ctx context.Context, id int, req UpdateBookRequest) (Book, error)
	Delete(ctx context.Context, id int) error
}
```

### 8.3 In-memory implementation **[I]**

**The concurrency note (important).** The server handles each request in its own goroutine, so the shared map is accessed concurrently. A bare map under concurrent read+write **panics** ("concurrent map writes"). Guard it with a `sync.RWMutex`: `RLock` for reads (many readers allowed), `Lock` for writes (exclusive). See `GO_GUIDE.md` §12 for the concurrency model behind this.

```go
type MemoryStore struct {
	mu     sync.RWMutex
	books  map[int]Book
	nextID int
}

func NewMemoryStore() *MemoryStore {
	return &MemoryStore{books: make(map[int]Book), nextID: 1}
}

func (s *MemoryStore) List(ctx context.Context) ([]Book, error) {
	s.mu.RLock()
	defer s.mu.RUnlock()
	out := make([]Book, 0, len(s.books))
	for _, b := range s.books {
		out = append(out, b)
	}
	return out, nil
}

func (s *MemoryStore) Get(ctx context.Context, id int) (Book, error) {
	s.mu.RLock()
	defer s.mu.RUnlock()
	b, ok := s.books[id]
	if !ok {
		return Book{}, ErrNotFound
	}
	return b, nil
}

func (s *MemoryStore) Create(ctx context.Context, req CreateBookRequest) (Book, error) {
	s.mu.Lock()
	defer s.mu.Unlock()
	b := Book{
		ID:        s.nextID,
		Title:     req.Title,
		Author:    req.Author,
		Year:      req.Year,
		CreatedAt: time.Now().UTC(),
	}
	s.books[s.nextID] = b
	s.nextID++
	return b, nil
}

func (s *MemoryStore) Update(ctx context.Context, id int, req UpdateBookRequest) (Book, error) {
	s.mu.Lock()
	defer s.mu.Unlock()
	b, ok := s.books[id]
	if !ok {
		return Book{}, ErrNotFound
	}
	// Only overwrite fields the client actually sent (non-nil pointer).
	if req.Title != nil {
		b.Title = *req.Title
	}
	if req.Author != nil {
		b.Author = *req.Author
	}
	if req.Year != nil {
		b.Year = *req.Year
	}
	s.books[id] = b
	return b, nil
}

func (s *MemoryStore) Delete(ctx context.Context, id int) error {
	s.mu.Lock()
	defer s.mu.Unlock()
	if _, ok := s.books[id]; !ok {
		return ErrNotFound
	}
	delete(s.books, id)
	return nil
}
```

### 8.4 The handler struct — dependency injection **[I]**

**The logic.** A handler struct holds the dependencies (the store, a logger) and exposes one method per endpoint. **Why:** no globals (testable, no hidden state), and each handler is a normal method you can call directly in a test (§17). The field is the **interface** type, so `main` injects the in-memory store now and a SQL store later with no handler changes.

```go
type BooksHandler struct {
	store BookStore // depend on the INTERFACE, not a concrete type
}

func NewBooksHandler(store BookStore) *BooksHandler {
	return &BooksHandler{store: store}
}

// Shared helpers (see §5.3 for the robust versions).
func writeJSON(w http.ResponseWriter, status int, v any) {
	w.Header().Set("Content-Type", "application/json; charset=utf-8")
	w.WriteHeader(status)
	_ = json.NewEncoder(w).Encode(v)
}
func writeError(w http.ResponseWriter, status int, msg string) {
	writeJSON(w, status, map[string]string{"error": msg})
}
func parseID(r *http.Request) (int, error) {
	id, err := strconv.Atoi(r.PathValue("id"))
	if err != nil || id <= 0 {
		return 0, fmt.Errorf("id must be a positive integer")
	}
	return id, nil
}
```

### 8.5 The endpoint methods **[I]**

```go
// GET /books
func (h *BooksHandler) List(w http.ResponseWriter, r *http.Request) {
	books, err := h.store.List(r.Context())
	if err != nil {
		writeError(w, http.StatusInternalServerError, "could not list books")
		return
	}
	writeJSON(w, http.StatusOK, map[string]any{"data": books, "count": len(books)})
}

// GET /books/{id}
func (h *BooksHandler) Get(w http.ResponseWriter, r *http.Request) {
	id, err := parseID(r)
	if err != nil {
		writeError(w, http.StatusBadRequest, err.Error())
		return
	}
	book, err := h.store.Get(r.Context(), id)
	if errors.Is(err, ErrNotFound) {
		writeError(w, http.StatusNotFound, fmt.Sprintf("book %d not found", id))
		return
	}
	if err != nil {
		writeError(w, http.StatusInternalServerError, "lookup failed")
		return
	}
	writeJSON(w, http.StatusOK, book)
}

// POST /books
func (h *BooksHandler) Create(w http.ResponseWriter, r *http.Request) {
	r.Body = http.MaxBytesReader(w, r.Body, 1<<20) // SECURITY: cap body
	defer r.Body.Close()

	var req CreateBookRequest
	dec := json.NewDecoder(r.Body)
	dec.DisallowUnknownFields()
	if err := dec.Decode(&req); err != nil {
		writeError(w, http.StatusBadRequest, "invalid JSON: "+err.Error())
		return
	}
	// Validate (see §10 for a structured validator).
	if errs := req.Validate(); len(errs) > 0 {
		writeJSON(w, http.StatusUnprocessableEntity, map[string]any{
			"error": "validation failed", "fields": errs,
		})
		return
	}
	book, err := h.store.Create(r.Context(), req)
	if err != nil {
		writeError(w, http.StatusInternalServerError, "could not create book")
		return
	}
	// 201 + Location header pointing at the new resource (REST convention).
	w.Header().Set("Location", fmt.Sprintf("/books/%d", book.ID))
	writeJSON(w, http.StatusCreated, book)
}

// PATCH /books/{id} (partial update)
func (h *BooksHandler) Update(w http.ResponseWriter, r *http.Request) {
	id, err := parseID(r)
	if err != nil {
		writeError(w, http.StatusBadRequest, err.Error())
		return
	}
	r.Body = http.MaxBytesReader(w, r.Body, 1<<20)
	defer r.Body.Close()

	var req UpdateBookRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		writeError(w, http.StatusBadRequest, "invalid JSON: "+err.Error())
		return
	}
	book, err := h.store.Update(r.Context(), id, req)
	if errors.Is(err, ErrNotFound) {
		writeError(w, http.StatusNotFound, fmt.Sprintf("book %d not found", id))
		return
	}
	if err != nil {
		writeError(w, http.StatusInternalServerError, "update failed")
		return
	}
	writeJSON(w, http.StatusOK, book)
}

// DELETE /books/{id}
func (h *BooksHandler) Delete(w http.ResponseWriter, r *http.Request) {
	id, err := parseID(r)
	if err != nil {
		writeError(w, http.StatusBadRequest, err.Error())
		return
	}
	if err := h.store.Delete(r.Context(), id); errors.Is(err, ErrNotFound) {
		writeError(w, http.StatusNotFound, fmt.Sprintf("book %d not found", id))
		return
	} else if err != nil {
		writeError(w, http.StatusInternalServerError, "delete failed")
		return
	}
	w.WriteHeader(http.StatusNoContent) // 204, no body
}
```

### 8.6 Wiring it together **[I]**

```go
func main() {
	store := NewMemoryStore()
	h := NewBooksHandler(store) // inject the in-memory store via the interface

	mux := http.NewServeMux()
	mux.HandleFunc("GET /books", h.List)
	mux.HandleFunc("POST /books", h.Create)
	mux.HandleFunc("GET /books/{id}", h.Get)
	mux.HandleFunc("PATCH /books/{id}", h.Update)
	mux.HandleFunc("DELETE /books/{id}", h.Delete)
	mux.HandleFunc("GET /health", func(w http.ResponseWriter, r *http.Request) {
		writeJSON(w, http.StatusOK, map[string]string{"status": "ok"})
	})

	// Apply global middleware (§7). Recoverer outermost.
	handler := Chain(mux, Recoverer, Logger)

	// SECURITY: always an explicit http.Server with timeouts (§14).
	srv := &http.Server{
		Addr:              ":8080",
		Handler:           handler,
		ReadHeaderTimeout: 5 * time.Second,
		ReadTimeout:       10 * time.Second,
		WriteTimeout:      15 * time.Second,
		IdleTimeout:       120 * time.Second,
		MaxHeaderBytes:    1 << 20,
	}
	log.Println("Books API on http://localhost:8080")
	log.Fatal(srv.ListenAndServe())
}
```

### 8.7 Exercise with curl **[B]**

```bash
# Create -> 201 with Location header
curl -i -X POST http://localhost:8080/books \
  -H "Content-Type: application/json" \
  -d '{"title":"The Go Programming Language","author":"Donovan & Kernighan","year":2015}'

curl http://localhost:8080/books          # list
curl http://localhost:8080/books/1        # get one
curl -X PATCH http://localhost:8080/books/1 -d '{"year":2016}'  # partial update
curl -i -X DELETE http://localhost:8080/books/1                 # -> 204
```

---

## 9. Error handling & JSON error responses

### 9.1 The problem and the architecture **[I]**

**The problem.** Scattering `http.Error(w, ..., 500)` through handlers leaks inconsistency: some errors are JSON, some plain text; some leak internal details; status codes drift. **The architecture** that fixes it: (1) the **store/service layer returns sentinel or typed errors** (`ErrNotFound`, a `ValidationError`), (2) **handlers translate** those into HTTP status + a consistent JSON shape, (3) **internal errors are logged but never echoed** to the client.

**The logic of sentinel vs typed errors** (see `GO_GUIDE.md` §10 for the language detail):

- **Sentinel** (`var ErrNotFound = errors.New(...)`): a known, comparable error checked with `errors.Is`. Good for "this specific known condition."
- **Typed** (a struct implementing `error`): carries *data* (which field failed, a conflict's existing id) checked with `errors.As`. Good when the handler needs details to build the response.

### 9.2 A typed API error you can map to status codes **[I]**

```go
// APIError carries everything a handler needs to render a response.
type APIError struct {
	Status  int    // HTTP status to send
	Code    string // stable machine code, e.g. "BOOK_NOT_FOUND"
	Message string // human-readable, safe to show the client
	err     error  // wrapped underlying error — logged, never sent
}

func (e *APIError) Error() string { return e.Message }
func (e *APIError) Unwrap() error { return e.err } // lets errors.Is/As see the cause

// Constructors keep call sites tidy.
func NotFound(code, msg string) *APIError {
	return &APIError{Status: http.StatusNotFound, Code: code, Message: msg}
}
func Internal(err error) *APIError {
	return &APIError{Status: http.StatusInternalServerError, Code: "INTERNAL",
		Message: "internal server error", err: err} // generic message; detail stays in err
}
```

### 9.3 A handler adapter that centralizes error translation **[I/A]**

**The logic / why this pattern is so loved in Go.** Instead of every handler repeating "if err != nil { write status }", let handlers **return an error**, and wrap them with an adapter that does the translation once. This is the "handlers return errors" pattern popularized by Go's blog. It removes boilerplate, guarantees consistency, and centralizes the "never leak internals" rule.

```go
// apiHandler is a handler that returns an error instead of writing it.
type apiHandler func(w http.ResponseWriter, r *http.Request) error

// ServeHTTP makes apiHandler satisfy http.Handler, translating any error.
func (h apiHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	if err := h(w, r); err != nil {
		var apiErr *APIError
		if !errors.As(err, &apiErr) {
			// An un-typed error => treat as 500 and log the detail.
			apiErr = Internal(err)
		}
		if apiErr.Status >= 500 {
			slog.Error("request failed", "path", r.URL.Path, "err", apiErr.err)
		}
		writeJSON(w, apiErr.Status, ErrorResponse{Error: apiErr.Message, Code: apiErr.Code})
	}
}

// Now a handler is clean: it just returns errors.
func getBook(store BookStore) apiHandler {
	return func(w http.ResponseWriter, r *http.Request) error {
		id, err := parseID(r)
		if err != nil {
			return &APIError{Status: http.StatusBadRequest, Code: "BAD_ID", Message: err.Error()}
		}
		book, err := store.Get(r.Context(), id)
		if errors.Is(err, ErrNotFound) {
			return NotFound("BOOK_NOT_FOUND", fmt.Sprintf("book %d not found", id))
		}
		if err != nil {
			return Internal(err) // logged, generic message to client
		}
		writeJSON(w, http.StatusOK, book)
		return nil
	}
}

// Register: mux.Handle("GET /books/{id}", getBook(store))
```

> **SECURITY recap:** the golden rule is **log details, return generics for 5xx.** A stack trace or DB error string in a response is an information-disclosure vulnerability (it reveals table names, file paths, library versions). The adapter above enforces this in one place.

---

## 10. Validation

### 10.1 Why validate, and where **[I]**

**What/why.** Decoding JSON only checks *shape and types*; it does not check that `year` is positive, `email` looks like an email, or `title` isn't 10 MB of text. Validation is enforcing your **business rules** on untrusted input. **Where:** validate **in the handler/service layer, right after decoding, before touching storage.** Return **422 Unprocessable Entity** with per-field messages so clients can show form errors.

**The stdlib stance.** Go's standard library has no validation framework (frameworks like Gin lean on `go-playground/validator` with struct tags). With `net/http` you typically write a small explicit `Validate()` method — verbose but dependency-free, fully under your control, and trivial to read.

### 10.2 An explicit validator returning field errors **[I]**

```go
import (
	"net/mail"
	"strings"
	"unicode/utf8"
)

// Validate checks business rules and returns a map of field -> message.
// An empty map means valid. Returning per-field errors gives clients a
// great UX (highlight the exact bad field).
func (r CreateBookRequest) Validate() map[string]string {
	errs := map[string]string{}

	title := strings.TrimSpace(r.Title)
	if title == "" {
		errs["title"] = "title is required"
	} else if utf8.RuneCountInString(title) > 200 { // count runes, not bytes
		errs["title"] = "title must be at most 200 characters"
	}

	if strings.TrimSpace(r.Author) == "" {
		errs["author"] = "author is required"
	}

	// A book year sanity range — adjust to your domain.
	if r.Year < 1450 || r.Year > time.Now().Year()+1 {
		errs["year"] = "year is out of range"
	}
	return errs
}

// Example of validating an email field with the stdlib (no regex needed):
func validEmail(s string) bool {
	_, err := mail.ParseAddress(s)
	return err == nil
}
```

> **Security & robustness notes:** (1) measure string lengths in **runes** (`utf8.RuneCountInString`) when you mean "characters" — bytes overcount multibyte UTF-8. (2) **`TrimSpace` before checking "required"** so a whitespace-only string doesn't sneak through. (3) Validate **maximum** lengths, not just presence — unbounded strings are a memory-DoS vector even after the body-size cap. (4) For emails, prefer `net/mail.ParseAddress` over a hand-rolled regex (email regexes are notoriously wrong).

### 10.3 Wiring validation into a handler **[I]**

```go
func (h *BooksHandler) Create(w http.ResponseWriter, r *http.Request) {
	r.Body = http.MaxBytesReader(w, r.Body, 1<<20)
	defer r.Body.Close()

	var req CreateBookRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		writeError(w, http.StatusBadRequest, "invalid JSON")
		return
	}
	if fieldErrs := req.Validate(); len(fieldErrs) > 0 {
		writeJSON(w, http.StatusUnprocessableEntity, ErrorResponse{
			Error: "validation failed", Code: "VALIDATION", Fields: fieldErrs,
		})
		return
	}
	book, _ := h.store.Create(r.Context(), req)
	writeJSON(w, http.StatusCreated, book)
}
```

---

## 11. Context — cancellation, deadlines, values

### 11.1 What context is and why HTTP needs it **[I]**

**What it is.** `context.Context` is Go's standard mechanism for three things: **cancellation** (tell downstream work to stop), **deadlines/timeouts** (auto-cancel after a duration), and **request-scoped values** (carry a user id or trace id without adding parameters to every function). (Language-level detail in `GO_GUIDE.md` §12.)

**Why HTTP needs it.** Every `*http.Request` *already carries a context* (`r.Context()`), and the server **cancels it automatically when the client disconnects.** So if you propagate `r.Context()` into your database calls, HTTP requests, and goroutines, then when a user closes the browser mid-request, your expensive DB query gets cancelled too — no wasted work. **Threading context everywhere is the single most impactful reliability habit in a Go service.**

### 11.2 Propagating the request context into I/O **[I]**

```go
func getUserHandler(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context() // cancelled if the client disconnects

	id := r.PathValue("id")
	// Pass ctx to EVERY I/O call. If the client leaves, the DB driver
	// aborts the in-flight query instead of running it to completion.
	row := db.QueryRowContext(ctx, "SELECT name FROM users WHERE id = $1", id)
	// ... scan row, respond ...
	_ = row
}
```

### 11.3 Adding a per-handler timeout **[I]**

**When:** you call a slow dependency and want to bound how long you'll wait. `context.WithTimeout` derives a child context that cancels after the duration *or* when the parent cancels — whichever comes first. **Always `defer cancel()`** to release the timer (a forgotten cancel is a goroutine/resource leak — `go vet` flags it).

```go
func slowHandler(w http.ResponseWriter, r *http.Request) {
	// 2s budget on top of the request's own lifetime.
	ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
	defer cancel() // ALWAYS — even on the success path

	result, err := doSlowWork(ctx)
	if err != nil {
		// Distinguish a timeout/cancel from a real error.
		if errors.Is(ctx.Err(), context.DeadlineExceeded) {
			http.Error(w, "request timed out", http.StatusGatewayTimeout)
			return
		}
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}
	writeJSON(w, http.StatusOK, result)
}
```

### 11.4 Carrying values through context — the typed-key pattern **[I/A]**

**The logic / why a private key type.** `context.WithValue(ctx, key, val)` stores a value; `ctx.Value(key)` reads it. The danger: if two packages both used the string `"userID"` as a key, they'd collide. The idiom is an **unexported custom key type** so no other package can accidentally (or maliciously) read or overwrite your value. Use context values *only* for request-scoped metadata (auth identity, request/trace id) — **never** for passing required function parameters (that hides dependencies).

```go
// Unexported key type: no other package can produce a value of this type,
// so collisions are impossible.
type ctxKey int

const userIDKey ctxKey = iota

// Auth middleware stores the authenticated user id in the context.
func WithUserID(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		userID := extractUserIDFromToken(r.Header.Get("Authorization"))
		ctx := context.WithValue(r.Context(), userIDKey, userID)
		// r.WithContext returns a SHALLOW COPY of r with the new context.
		next.ServeHTTP(w, r.WithContext(ctx))
	})
}

// Typed accessor — handlers call this, never ctx.Value directly.
func UserIDFromCtx(ctx context.Context) (string, bool) {
	id, ok := ctx.Value(userIDKey).(string)
	return id, ok
}

func meHandler(w http.ResponseWriter, r *http.Request) {
	userID, ok := UserIDFromCtx(r.Context())
	if !ok {
		http.Error(w, "not authenticated", http.StatusUnauthorized)
		return
	}
	fmt.Fprintf(w, "hello user %s\n", userID)
}
```

### 11.5 Detecting client disconnect while streaming **[A]**

**When:** long-lived responses (Server-Sent Events, large exports, progress streams). Watch `ctx.Done()` and stop producing work the moment the client leaves; `http.Flusher` pushes buffered bytes to the client immediately.

```go
func streamHandler(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()
	flusher, ok := w.(http.Flusher) // not all writers support Flush
	if !ok {
		http.Error(w, "streaming unsupported", http.StatusInternalServerError)
		return
	}
	w.Header().Set("Content-Type", "text/event-stream")

	for i := 0; i < 100; i++ {
		select {
		case <-ctx.Done():
			// Client disconnected (or timeout): stop doing work immediately.
			slog.Info("client gone, stopping stream", "err", ctx.Err())
			return
		default:
			fmt.Fprintf(w, "data: chunk %d\n\n", i)
			flusher.Flush() // send this chunk now, don't buffer
			time.Sleep(100 * time.Millisecond)
		}
	}
}
```

---

## 12. Structured logging with `log/slog`

### 12.1 Why structured logging **[I]**

⚡ **Version note:** `log/slog` is in the stdlib since **Go 1.21** — no third-party logger needed. **What it is:** a logger that emits **key/value** records (JSON or text) at **levels** (Debug/Info/Warn/Error). **Why it beats `log.Printf`:** machine-parseable output that log aggregators (Loki, Elasticsearch, CloudWatch) can index and query — you can filter by `status>=500` or `user_id=123` instead of grepping free text. **Best practices:** JSON handler in production (text for local dev), set the level from config/env, and **never log secrets** (tokens, passwords, full auth headers).

```go
package main

import (
	"log/slog"
	"net/http"
	"os"
)

func main() {
	// JSON for prod (machine-readable). Use NewTextHandler for local dev.
	logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
		Level: slog.LevelInfo, // raise to LevelDebug locally
	}))
	slog.SetDefault(logger) // so package-level slog.Info uses it

	slog.Info("server starting", "addr", ":8080") // key/value pairs

	mux := http.NewServeMux()
	mux.HandleFunc("GET /ping", func(w http.ResponseWriter, r *http.Request) {
		slog.Info("ping", "remote", r.RemoteAddr)
		w.Write([]byte("pong"))
	})
	http.ListenAndServe(":8080", mux)
}
```

Output:

```json
{"time":"2026-06-24T10:00:00Z","level":"INFO","msg":"server starting","addr":":8080"}
{"time":"2026-06-24T10:00:01Z","level":"INFO","msg":"ping","remote":"127.0.0.1:54321"}
```

### 12.2 Per-request logger with a request ID (context-carried) **[I/A]**

**The logic.** Attach a unique **request id** to each request and put a logger pre-loaded with it into the context. Every log line from that request then shares the id, so you can trace one request across many log lines — invaluable in production debugging. This combines §11 (context values) with §7 (middleware) and §12 (slog children via `With`).

```go
import "crypto/rand"
import "encoding/hex"

type ctxLoggerKey struct{}

// RequestLogger injects a request-scoped logger carrying a unique request id.
func RequestLogger(base *slog.Logger) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			reqID := newRequestID()
			w.Header().Set("X-Request-Id", reqID) // surface it to clients/proxies
			// .With returns a child logger that includes these fields on every line.
			l := base.With("request_id", reqID, "method", r.Method, "path", r.URL.Path)
			ctx := context.WithValue(r.Context(), ctxLoggerKey{}, l)
			next.ServeHTTP(w, r.WithContext(ctx))
		})
	}
}

// LoggerFromCtx returns the request logger, falling back to the default.
func LoggerFromCtx(ctx context.Context) *slog.Logger {
	if l, ok := ctx.Value(ctxLoggerKey{}).(*slog.Logger); ok {
		return l
	}
	return slog.Default()
}

func newRequestID() string {
	b := make([]byte, 8)
	_, _ = rand.Read(b)
	return hex.EncodeToString(b)
}

// In a handler:
//   log := LoggerFromCtx(r.Context())
//   log.Info("created book", "book_id", book.ID)  // includes request_id automatically
```

---

## 13. Graceful shutdown & signal handling

### 13.1 Why graceful shutdown matters **[I/A]**

**The problem.** When you stop a server (Ctrl+C, a `SIGTERM` from Kubernetes during a deploy), an abrupt exit **drops every in-flight request** — half-written responses, interrupted DB transactions, angry clients. **Graceful shutdown** instead: (1) stops accepting *new* connections, (2) lets *active* handlers finish (up to a deadline), then (3) exits cleanly. **The mechanism:** run the server in a goroutine, block `main` on an OS signal via `signal.NotifyContext`, then call `srv.Shutdown(ctx)` with a bounded timeout.

⚡ **Version note:** `signal.NotifyContext` (Go 1.16+) gives you a context that's cancelled on the chosen signals — much cleaner than the old `make(chan os.Signal)` dance. `srv.Shutdown` returns `http.ErrServerClosed` from `ListenAndServe`, which you must treat as the *normal* close, not an error.

```go
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
)

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("GET /slow", func(w http.ResponseWriter, r *http.Request) {
		time.Sleep(3 * time.Second) // simulate in-flight work
		w.Write([]byte("done"))
	})

	srv := &http.Server{
		Addr:              ":8080",
		Handler:           mux,
		ReadHeaderTimeout: 5 * time.Second,
		ReadTimeout:       10 * time.Second,
		WriteTimeout:      15 * time.Second,
		IdleTimeout:       120 * time.Second,
	}

	// 1. Run the server in a goroutine so main can wait for a signal.
	go func() {
		slog.Info("server started", "addr", srv.Addr)
		// ErrServerClosed is the EXPECTED error after Shutdown — not a failure.
		if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
			slog.Error("listen failed", "err", err)
			os.Exit(1)
		}
	}()

	// 2. Block until SIGINT (Ctrl+C) or SIGTERM (orchestrator stop).
	ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
	defer stop()
	<-ctx.Done()
	stop() // restore default handling so a SECOND Ctrl+C force-kills

	slog.Info("shutting down, draining in-flight requests...")

	// 3. Give active handlers up to 15s to finish, then force-close.
	shutdownCtx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
	defer cancel()
	if err := srv.Shutdown(shutdownCtx); err != nil {
		slog.Error("graceful shutdown failed; forcing close", "err", err)
		_ = srv.Close() // last resort
		os.Exit(1)
	}
	slog.Info("shutdown complete")
}
```

**What happens on Ctrl+C, step by step:** `ctx.Done()` unblocks → `Shutdown` stops the listener (no new connections) → it waits for active handlers (the `/slow` request finishes its 3s) → if everything drains before 15s, the process exits 0; if not, the timeout fires and you `Close()` and exit non-zero.

> **Windows note:** `SIGTERM` exists in Go's `syscall` on Windows but is not delivered the way Unix delivers it; Ctrl+C produces `SIGINT`. For services, the lifecycle is managed by the Service Control Manager. In containers (Linux), `SIGTERM` is what Docker/Kubernetes send — handle it.

---

## 14. Security hardening

This section consolidates the security practices scattered through the guide into a checklist with the reasoning behind each. **Security is not a feature you add at the end** — these are defaults you set from line one.

### 14.1 Always set server timeouts (defeat slowloris) **[I/A]**

**The threat.** A "slowloris" attack opens many connections and sends headers/body **one byte at a time, very slowly**, holding your connection slots open forever and starving real clients. The `http.ListenAndServe` convenience form has **no timeouts**, so it is vulnerable by default. **The fix:** an explicit `http.Server` with every timeout set.

| Field | What it bounds | Why |
|---|---|---|
| `ReadHeaderTimeout` | Time to read **request headers** | The primary slowloris defense |
| `ReadTimeout` | Time to read headers **+ body** | Caps total inbound read time |
| `WriteTimeout` | Time to write the **response** | Caps slow-reading clients |
| `IdleTimeout` | Keep-alive idle time before close | Frees idle connection slots |
| `MaxHeaderBytes` | Max total header size | Caps header-bomb memory use |

```go
srv := &http.Server{
	Addr:              ":8080",
	Handler:           handler,
	ReadHeaderTimeout: 5 * time.Second,   // MUST set — slowloris defense
	ReadTimeout:       10 * time.Second,
	WriteTimeout:      15 * time.Second,
	IdleTimeout:       120 * time.Second,
	MaxHeaderBytes:    1 << 20,           // 1 MiB of headers max
}
```

### 14.2 Limit request body size **[I]**

Covered in §4.4: wrap `r.Body` with `http.MaxBytesReader(w, r.Body, n)` **before** decoding. Without it, a client can stream an unbounded body and exhaust memory. Set the cap per-endpoint (uploads need more than a JSON POST).

### 14.3 Security response headers **[I]**

**What:** a middleware that sets defensive headers on every response. **Why each:**

- `X-Content-Type-Options: nosniff` — stop browsers from MIME-sniffing a response into something executable.
- `X-Frame-Options: DENY` — prevent clickjacking by disallowing your responses in frames.
- `Strict-Transport-Security` — force HTTPS for future visits (only over real HTTPS).
- `Content-Security-Policy` — for HTML responses, restrict what can load (the strongest XSS defense).
- `Referrer-Policy` — limit referrer leakage.

```go
func SecurityHeaders(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		h := w.Header()
		h.Set("X-Content-Type-Options", "nosniff")
		h.Set("X-Frame-Options", "DENY")
		h.Set("Referrer-Policy", "no-referrer")
		// Only meaningful (and only set) when served over HTTPS:
		if r.TLS != nil {
			h.Set("Strict-Transport-Security", "max-age=63072000; includeSubDomains")
		}
		next.ServeHTTP(w, r)
	})
}
```

### 14.4 TLS (HTTPS) **[I/A]**

**Why:** HTTP is plaintext — credentials and data are readable by anyone on the path. Always serve real traffic over TLS. **How:** `ListenAndServeTLS(addr, certFile, keyFile, handler)`, or set `srv.TLSConfig` for fine control. In production, TLS is often **terminated at a load balancer / reverse proxy** (nginx, Caddy, a cloud LB), and your Go server speaks plain HTTP *on a private network* behind it — that's fine and common. For Go-terminated TLS, harden the config: require TLS 1.2+ and prefer modern cipher suites.

```go
srv := &http.Server{
	Addr:    ":443",
	Handler: handler,
	TLSConfig: &tls.Config{
		MinVersion: tls.VersionTLS12, // never allow TLS 1.0/1.1
	},
	ReadHeaderTimeout: 5 * time.Second,
}
// cert.pem / key.pem from your CA (or mkcert / Let's Encrypt / autocert).
log.Fatal(srv.ListenAndServeTLS("cert.pem", "key.pem"))
```

> **Tip:** for automatic Let's Encrypt certificates, `golang.org/x/crypto/acme/autocert` provides a `GetCertificate` you plug into `TLSConfig` — zero-touch HTTPS.

### 14.5 Input validation & injection defense **[I/A]**

- **Validate everything** (§10): types, ranges, lengths, formats. Reject early with 400/422.
- **SQL injection:** *always* use parameterized queries (`db.QueryContext(ctx, "... WHERE id = $1", id)`). **Never** build SQL by string concatenation with request data. (See `GO_ENT_ORM_GUIDE.md`.)
- **Path traversal:** when a request value becomes a filename, `filepath.Base` it and confirm the resolved path stays inside your intended directory.
- **Mass assignment:** decode into input DTOs that *omit* protected fields (§6), so a client can't set `is_admin` or `id`.
- **Don't reflect untrusted input into HTML** without `html/template` (which auto-escapes) — that's XSS.

### 14.6 The trusted-proxy / client IP caveat **[I]**

`r.RemoteAddr` is the **direct peer**. Behind a load balancer that's the LB's IP, not the user's. The real client may be in `X-Forwarded-For` — but **that header is client-spoofable**, so only trust it when the request *definitely* came through a proxy you control (validate the immediate peer is your proxy, or use the proxy's signed header). Naively trusting `X-Forwarded-For` lets attackers forge IPs to bypass rate limits or audit logs.

### 14.7 Security checklist **[I/A]**

- [ ] Explicit `http.Server` with all timeouts + `MaxHeaderBytes` set
- [ ] `http.MaxBytesReader` on every body-reading handler
- [ ] TLS in front (terminated by you or a proxy); TLS 1.2+ minimum
- [ ] Security headers middleware
- [ ] CORS allow-list (never `*` with credentials)
- [ ] Auth via constant-time comparison; identity in context
- [ ] Rate limiting on public/auth endpoints; clamp pagination limits
- [ ] Parameterized DB queries; `filepath.Base` for upload names; DTOs for input
- [ ] 5xx errors logged, never echoed to the client
- [ ] No secrets in logs

---

## 15. `http.Client` — making outbound requests

### 15.1 Never use `http.DefaultClient` in production **[I]**

**The trap.** `http.Get`, `http.Post`, and `http.DefaultClient` have **no timeout.** One slow or hostile upstream blocks the calling goroutine *forever*; under load that exhausts goroutines and takes your service down. **Always construct a client with an explicit `Timeout`** (and ideally a tuned `Transport`). **Reuse one client** across requests — it pools connections; creating a new client per call defeats keep-alive and leaks connections.

```go
import (
	"net"
	"net/http"
	"time"
)

// One shared, tuned client for the whole process.
var apiClient = &http.Client{
	Timeout: 10 * time.Second, // TOTAL budget: connect + send + read body
	Transport: &http.Transport{
		// Tuning the pool prevents connection exhaustion under load.
		MaxIdleConns:        100,
		MaxIdleConnsPerHost: 10,
		IdleConnTimeout:     90 * time.Second,
		DialContext: (&net.Dialer{
			Timeout:   3 * time.Second, // TCP connect timeout
			KeepAlive: 30 * time.Second,
		}).DialContext,
		TLSHandshakeTimeout: 3 * time.Second,
	},
}
```

### 15.2 GET with proper body handling **[I]**

**The must-dos:** always pass a context, always `defer resp.Body.Close()` (even if you don't read it — otherwise the connection can't be reused and leaks), check the status code, and drain the body on early return so the connection returns to the pool.

```go
func fetchUser(ctx context.Context, id string) (map[string]any, error) {
	req, err := http.NewRequestWithContext(ctx, http.MethodGet,
		"https://api.example.com/users/"+id, nil)
	if err != nil {
		return nil, err
	}
	resp, err := apiClient.Do(req)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close() // ALWAYS

	if resp.StatusCode != http.StatusOK {
		// Drain so the connection can be reused, then report.
		io.Copy(io.Discard, resp.Body)
		return nil, fmt.Errorf("upstream returned %d", resp.StatusCode)
	}
	var result map[string]any
	if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
		return nil, err
	}
	return result, nil
}
```

### 15.3 POST JSON with context **[I]**

```go
func postJSON(ctx context.Context, url string, payload any) (*http.Response, error) {
	body, err := json.Marshal(payload)
	if err != nil {
		return nil, err
	}
	req, err := http.NewRequestWithContext(ctx, http.MethodPost, url, bytes.NewReader(body))
	if err != nil {
		return nil, err
	}
	req.Header.Set("Content-Type", "application/json")
	return apiClient.Do(req)
}
```

> **Propagate the inbound request's context to outbound calls** (`fetchUser(r.Context(), id)`). Then if the original client disconnects, your upstream call is cancelled too — no orphaned work. **Security:** validate/allow-list URLs you fetch on behalf of users (SSRF defense) — never let a user-supplied URL reach internal addresses (`169.254.169.254`, `localhost`, private ranges).

---

## 16. Serving static files & embedding

### 16.1 Basic file server **[I]**

**What:** `http.FileServer` serves a directory tree; `http.StripPrefix` removes the URL prefix before the file server resolves the path (otherwise it would look for `./public/static/...`).

```go
mux := http.NewServeMux()
fs := http.FileServer(http.Dir("./public"))
// Request /static/css/app.css -> file ./public/css/app.css
mux.Handle("/static/", http.StripPrefix("/static/", fs))
```

> **Security:** `http.FileServer` already cleans paths and blocks `..` traversal, and it won't serve files outside the root — but it *does* list directory contents by default. To disable directory listings, wrap the FS or serve specific files. Never point a file server at a directory containing secrets (`.env`, keys, `.git`).

### 16.2 Embedding files in the binary with `//go:embed` **[I]**

⚡ **Version note:** `embed` is Go 1.16+. **Why:** ship static assets *inside* the single binary — no separate `public/` folder to deploy, no path issues, atomic deploys. Ideal for a small SPA frontend or default templates.

```go
package main

import (
	"embed"
	"io/fs"
	"net/http"
)

//go:embed public
var publicFiles embed.FS

func main() {
	// The embed includes the "public/" prefix; fs.Sub re-roots to it so URLs
	// don't need /public in them.
	sub, _ := fs.Sub(publicFiles, "public")
	mux := http.NewServeMux()
	mux.Handle("/static/", http.StripPrefix("/static/", http.FileServer(http.FS(sub))))
	http.ListenAndServe(":8080", mux)
}
```

### 16.3 SPA catch-all (serve index.html for unknown routes) **[I]**

**When:** a single-page app (React/Vue) where the client router owns paths — the server must return `index.html` for any non-asset path so deep links work on refresh.

```go
func spaHandler(dir string) http.HandlerFunc {
	fileServer := http.FileServer(http.Dir(dir))
	return func(w http.ResponseWriter, r *http.Request) {
		// Use filepath.Clean + Join to avoid traversal when probing for a file.
		path := filepath.Join(dir, filepath.Clean("/"+r.URL.Path))
		if info, err := os.Stat(path); err == nil && !info.IsDir() {
			fileServer.ServeHTTP(w, r) // real asset exists -> serve it
			return
		}
		http.ServeFile(w, r, filepath.Join(dir, "index.html")) // fall back to SPA shell
	}
}

// mux.Handle("/", spaHandler("./dist"))
```

---

## 17. Testing handlers with `net/http/httptest`

### 17.1 Why httptest and the two core types **[I]**

**What it is.** `net/http/httptest` lets you test handlers **without binding a real TCP port** — fast, deterministic, parallelizable. The two workhorses: `httptest.NewRequest` (build a fake `*http.Request`) and `httptest.NewRecorder` (a `ResponseWriter` that records what the handler wrote so you can assert on it). For full-stack tests, `httptest.NewServer` spins up a real server on an OS-assigned port.

**Best practice:** because your handlers depend on an *interface* store (§8.2), tests inject a fresh in-memory store (or a mock) — no database required, total isolation between tests.

### 17.2 Testing a handler directly **[I]**

```go
package main

import (
	"context"
	"encoding/json"
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
)

func TestCreateBook(t *testing.T) {
	store := NewMemoryStore()       // fresh, isolated state
	h := NewBooksHandler(store)

	body := `{"title":"Test Book","author":"Test Author","year":2024}`
	req := httptest.NewRequest(http.MethodPost, "/books", strings.NewReader(body))
	req.Header.Set("Content-Type", "application/json")

	rec := httptest.NewRecorder() // records the response
	h.Create(rec, req)            // call the handler method directly

	if rec.Code != http.StatusCreated {
		t.Fatalf("status: want 201, got %d (body: %s)", rec.Code, rec.Body.String())
	}
	var got Book
	if err := json.NewDecoder(rec.Body).Decode(&got); err != nil {
		t.Fatal("decode:", err)
	}
	if got.Title != "Test Book" {
		t.Errorf("title: want %q, got %q", "Test Book", got.Title)
	}
	if got.ID == 0 {
		t.Error("expected a non-zero ID")
	}
}
```

### 17.3 Testing routes that use path values (Go 1.22+) **[I]**

**The gotcha:** `r.PathValue("id")` is populated by the **mux**, not by `httptest.NewRequest`. So to test a `{id}` route either (a) serve the request through a mux that has the pattern registered, or (b) use `req.SetPathValue("id", "1")` (Go 1.22+) to set it directly.

```go
func TestGetBook(t *testing.T) {
	store := NewMemoryStore()
	store.Create(context.Background(), CreateBookRequest{Title: "Go in Action", Author: "Kennedy", Year: 2015})
	h := NewBooksHandler(store)

	// Option A: route through a mux so PathValue is populated.
	mux := http.NewServeMux()
	mux.HandleFunc("GET /books/{id}", h.Get)

	req := httptest.NewRequest(http.MethodGet, "/books/1", nil)
	rec := httptest.NewRecorder()
	mux.ServeHTTP(rec, req) // use the mux, not h.Get directly

	if rec.Code != http.StatusOK {
		t.Errorf("want 200, got %d", rec.Code)
	}
}

func TestGetBook_SetPathValue(t *testing.T) {
	store := NewMemoryStore()
	store.Create(context.Background(), CreateBookRequest{Title: "X", Author: "Y", Year: 2020})
	h := NewBooksHandler(store)

	req := httptest.NewRequest(http.MethodGet, "/books/1", nil)
	req.SetPathValue("id", "1") // Go 1.22+ — set the wildcard without a mux
	rec := httptest.NewRecorder()
	h.Get(rec, req)

	if rec.Code != http.StatusOK {
		t.Errorf("want 200, got %d", rec.Code)
	}
}
```

### 17.4 Table-driven tests — the idiomatic Go style **[I/A]**

**Why:** one test function, many cases, each with a name — the canonical Go testing pattern (see `GO_GUIDE.md` §15). Great for covering status codes across many inputs.

```go
func TestGetBookStatuses(t *testing.T) {
	store := NewMemoryStore()
	store.Create(context.Background(), CreateBookRequest{Title: "A", Author: "B", Year: 2000})
	h := NewBooksHandler(store)
	mux := http.NewServeMux()
	mux.HandleFunc("GET /books/{id}", h.Get)

	cases := []struct {
		name   string
		path   string
		want   int
	}{
		{"existing", "/books/1", http.StatusOK},
		{"missing", "/books/999", http.StatusNotFound},
		{"non-numeric id", "/books/abc", http.StatusBadRequest},
	}
	for _, tc := range cases {
		t.Run(tc.name, func(t *testing.T) {
			req := httptest.NewRequest(http.MethodGet, tc.path, nil)
			rec := httptest.NewRecorder()
			mux.ServeHTTP(rec, req)
			if rec.Code != tc.want {
				t.Errorf("%s: want %d, got %d", tc.path, tc.want, rec.Code)
			}
		})
	}
}
```

### 17.5 Full-stack test with a real server **[A]**

`httptest.NewServer` is best for testing middleware chains and the real client/server round trip.

```go
func TestServerRoundTrip(t *testing.T) {
	store := NewMemoryStore()
	h := NewBooksHandler(store)
	mux := http.NewServeMux()
	mux.HandleFunc("GET /books", h.List)

	srv := httptest.NewServer(Chain(mux, Recoverer, Logger)) // exercise middleware too
	defer srv.Close()

	resp, err := http.Get(srv.URL + "/books")
	if err != nil {
		t.Fatal(err)
	}
	defer resp.Body.Close()
	if resp.StatusCode != http.StatusOK {
		t.Errorf("want 200, got %d", resp.StatusCode)
	}
}
```

```bash
go test ./...                 # all packages
go test -v ./...              # verbose
go test -run TestGetBook ./...# matching tests only
go test -race ./...           # CRITICAL for servers: detects data races
go test -cover ./...          # coverage
```

> **Run `go test -race` for any concurrent server.** It instruments memory access and flags data races (e.g. an unguarded shared map) that are otherwise nearly impossible to find. CI should always use it.

---

## 18. Structuring a real service

### 18.1 The layout and its rationale **[I/A]**

**The logic.** Start flat; grow into structure. The widely-used layout below separates **entry point** (`cmd/`), **private application code** (`internal/`, which Go *enforces* cannot be imported by other modules), and **responsibility per package**. Keep `main` thin (just wiring) so business logic is testable without starting a server.

```
booksapi/
├── cmd/
│   └── api/
│       └── main.go            # entry point: load config, wire deps, run server
├── internal/
│   ├── http/                  # transport layer
│   │   ├── handler.go         # BooksHandler + methods
│   │   ├── middleware.go      # Logger, Recoverer, CORS, Auth, SecurityHeaders
│   │   ├── routes.go          # builds the *http.ServeMux
│   │   └── respond.go         # writeJSON / writeError / error mapping
│   ├── book/                  # domain: model, DTOs, validation, service
│   │   ├── book.go            # Book, CreateBookRequest, Validate()
│   │   └── service.go         # business rules over the store
│   └── store/                 # persistence
│       ├── store.go           # the BookStore interface + ErrNotFound
│       ├── memory.go          # in-memory implementation
│       └── postgres.go        # SQL implementation (later)
├── go.mod
└── go.sum
```

### 18.2 The wiring file (`cmd/api/main.go`) **[I/A]**

This brings together everything: config from env, dependency construction via interfaces, route building, middleware chain, the hardened server, and graceful shutdown.

```go
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

	"booksapi/internal/http"   // your transport package (aliased below in real code)
	"booksapi/internal/store"
)

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
	slog.SetDefault(logger)

	// 1. Construct dependencies behind interfaces (swap memory<->postgres here).
	bookStore := store.NewMemoryStore()
	handler := httpx.NewBooksHandler(bookStore)

	// 2. Build routes + middleware.
	mux := httpx.Routes(handler)
	root := httpx.Chain(mux,
		httpx.Recoverer,
		httpx.RequestLogger(logger),
		httpx.SecurityHeaders,
		httpx.CORS("https://app.example.com"),
	)

	// 3. Hardened server (§14).
	addr := getenv("ADDR", ":8080")
	srv := &http.Server{
		Addr:              addr,
		Handler:           root,
		ReadHeaderTimeout: 5 * time.Second,
		ReadTimeout:       10 * time.Second,
		WriteTimeout:      15 * time.Second,
		IdleTimeout:       120 * time.Second,
		MaxHeaderBytes:    1 << 20,
	}

	// 4. Run + graceful shutdown (§13).
	go func() {
		logger.Info("listening", "addr", addr)
		if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
			logger.Error("server error", "err", err)
			os.Exit(1)
		}
	}()
	ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
	defer stop()
	<-ctx.Done()

	shutdownCtx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
	defer cancel()
	if err := srv.Shutdown(shutdownCtx); err != nil {
		logger.Error("shutdown error", "err", err)
	}
	logger.Info("bye")
}

func getenv(key, fallback string) string {
	if v := os.Getenv(key); v != "" {
		return v
	}
	return fallback
}
```

### 18.3 Structure rules of thumb **[I/A]**

| Rule | Why |
|---|---|
| `cmd/<name>/main.go` per binary | Multiple binaries live in separate `cmd/` dirs |
| `internal/` for app code | Go *forbids* external modules importing it |
| Depend on **interfaces**, inject concretes in `main` | Swappable storage, mockable tests |
| Keep `main` thin (wiring only) | Business logic stays testable without a server |
| Package by **responsibility**, not by type | `store`, `book`, `http` — not `models`, `controllers` |
| Avoid `util`/`helpers` packages | They become dumping grounds; name by what they do |
| Config from env/flags, not hard-coded | 12-factor; same binary across environments |

---

## 19. Tips, tricks & gotchas

1. **Always close `r.Body`.** Even when you `Decode` from it, `defer r.Body.Close()` so the connection can be reused.

2. **Always close *and drain* client response bodies.** `defer resp.Body.Close()` then `io.Copy(io.Discard, resp.Body)` on early returns — otherwise keep-alive can't reuse the connection.

3. **Never use `http.DefaultClient`/`http.Get` in production** — no timeout, blocks forever on a slow upstream (§15).

4. **Always set server timeouts** — the convenience `ListenAndServe` has none and is slowloris-vulnerable (§14.1).

5. **Never use the default mux** — global state; libraries can inject routes via `init()`. Use `http.NewServeMux()` (§3.5).

6. **`WriteHeader` takes effect once.** Set all headers *before* the first `Write`/`WriteHeader`. Setting a header after writing the body is silently dropped (§2.4).

7. **`json.NewEncoder(w).Encode(v)` appends a trailing `\n`**; `json.Marshal` does not. Harmless for APIs, but it surprises byte-exact test comparisons.

8. **`r.PathValue` returns `""` for a missing wildcard** — always validate; don't assume non-empty (§4.1).

9. **HTTP methods in patterns are case-sensitive and must be uppercase.** `"get /x"` ≠ `"GET /x"`. Use uppercase per spec (§3).

10. **Trailing slash = subtree.** `GET /api/v1/` matches everything under it and auto-301-redirects `/api/v1` → `/api/v1/`; `GET /api/v1` matches only the exact path. Use `{$}` to anchor (§3.2).

11. **A panic in a handler only kills that goroutine** but drops the client's connection abruptly. Use `Recoverer` middleware in production (§7.3).

12. **Use struct tags deliberately:** `json:"-"` to never serialize a field (passwords!), `json:"name,omitempty"` to drop zero values, snake_case names for API conventions.
    ```go
    type User struct {
    	ID       int    `json:"id"`
    	Name     string `json:"name"`
    	Password string `json:"-"`               // NEVER serialized
    	Bio      string `json:"bio,omitempty"`    // omitted when empty
    }
    ```

13. **`http.Error` sends `text/plain` + a newline.** Fine for quick errors, but use a JSON helper for consistent API error bodies (§5.3).

14. **Check `ctx.Err()` after context-aware calls** to distinguish "client cancelled / timed out" from a genuine failure, and return 504/499 vs 500 accordingly (§11.3).

15. **Don't store request-scoped *required* values only in context.** Context values are for cross-cutting metadata (auth id, trace id), not for smuggling mandatory parameters — that hides dependencies (§11.4).

16. **`time.Now().UTC()`** for stored timestamps — store and serialize in UTC; convert to local only at display time.

17. **Run `go test -race`** on any server; data races on shared maps/state are otherwise invisible until production (§17.5).

18. **Prefer `errors.Is`/`errors.As`** over `==`/type switches for error checks — they see through `fmt.Errorf("...: %w", err)` wrapping (§9).

---

## 20. Study path & build-to-learn projects

Work through these in order; each phase builds on the last. Build every mini-project — reading is not learning.

### Phase 1 — Core mechanics (1–2 days) **[B]**

1. **Hello server** — `http.HandleFunc`, explicit `http.Server`, test with curl (§2).
2. **The interface** — internalize "everything is an `http.Handler`"; write one struct handler and one `HandlerFunc` (§2.2–2.3).
3. **Routing** — an explicit `ServeMux`; method patterns, `{id}`, `{path...}`, `{$}`; `r.PathValue` (§3).
4. **Reading requests** — path/query params, JSON decode with size cap + `DisallowUnknownFields`, headers (§4).
5. **Writing responses** — status constants, header ordering, a `writeJSON` helper (§5).

> **Mini-project:** a "ping/pong" API with a JSON `/health`, one `{id}` route, and a `?q=` search route that clamps `limit`.

### Phase 2 — Building the API (2–3 days) **[I]**

6. **REST design** — model a resource, split DTOs from the model (§6).
7. **CRUD** — full Books API with a `sync.RWMutex` in-memory store (§8.1–8.6).
8. **Middleware** — write `Logger` (with status capture), `Recoverer`, `CORS`; understand the wrapper pattern and chaining (§7).
9. **Errors & validation** — sentinel + typed errors, the "handlers return errors" adapter, a field-level validator returning 422 (§9–§10).

> **Mini-project:** a "notes" CRUD API with full middleware, structured errors, and validation.

### Phase 3 — Production concerns (2–4 days) **[I/A]**

10. **Context** — thread `r.Context()` into all I/O, add per-handler timeouts, carry an auth id with a typed key (§11).
11. **Structured logging** — `log/slog` JSON logger + a request-id middleware (§12).
12. **Graceful shutdown** — `signal.NotifyContext` + `srv.Shutdown` (§13).
13. **Security hardening** — set all timeouts, body limits, security headers, CORS allow-list, rate limiter; work the §14.7 checklist.
14. **Interface-based store** — refactor handlers to depend on `BookStore`; add a second (mock or SQL) implementation (§8.2).

> **Mini-project:** upgrade "notes" with slog + request ids, graceful shutdown, full timeout/header hardening, and an interface store with two implementations.

### Phase 4 — Real-world additions (3–5 days) **[A]**

15. **Testing** — handler tests, `SetPathValue`, table-driven cases, `httptest.NewServer`, `-race`; aim for >80% coverage (§17).
16. **`http.Client`** — proxy/fetch a third-party API with a tuned, shared client and context propagation; add SSRF allow-listing (§15).
17. **Static & embed** — serve an embedded SPA with a catch-all (§16).
18. **Project structure** — split into `cmd/`, `internal/http`, `internal/book`, `internal/store` (§18).
19. **Cross over to Gin** — re-implement one resource in Gin and compare; you'll recognize every concept. See `GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md`.

> **Capstone — a "task manager" REST API** with: full CRUD `/tasks`; Bearer-token auth (constant-time, identity in context); `log/slog` + request ids; validation → 422; rate limiting + security headers + all timeouts; graceful shutdown; an interface store (memory + tests, optionally SQL via `GO_ENT_ORM_GUIDE.md`); `httptest` coverage with `-race`; an embedded static frontend; and an `http.Client` integration (e.g. fetch weather for a task's location) with SSRF protection.

### Reference: standard-library packages you'll use **[I]**

| Package | Purpose |
|---|---|
| `net/http` | HTTP server + client + router |
| `net/http/httptest` | HTTP testing utilities |
| `encoding/json` | JSON encode/decode |
| `context` | Cancellation, deadlines, request values |
| `log/slog` | Structured logging (1.21+) |
| `os/signal` + `syscall` | Signal handling for graceful shutdown |
| `crypto/subtle` | Constant-time comparison (auth) |
| `crypto/tls` | TLS configuration |
| `embed` | Embed assets in the binary (1.16+) |
| `strconv` | String ↔ number conversion |
| `sync` | `RWMutex` for shared state |
| `time` | Durations, timeouts, timestamps |
| `errors` | `Is`, `As`, `New`, `%w` wrapping |
| `net/mail` | Email parsing/validation |
| `path/filepath` | Safe path handling (traversal defense) |
| `golang.org/x/time/rate` | Token-bucket rate limiting (official `x/`) |

---

> **Final note.** `net/http` is genuinely enough for the vast majority of REST APIs — and even when you adopt Gin or another framework, *everything* you learned here still applies, because they are built on these primitives. Master the standard library first: the handler interface, the enhanced router, middleware composition, context propagation, graceful shutdown, and the security defaults. Those fundamentals are stable, portable across every Go web stack, and will serve you for years. When you're ready for the higher-level ergonomics, read `GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md` — and you'll see the same ideas wearing nicer clothes.
