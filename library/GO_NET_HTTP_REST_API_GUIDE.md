# Go `net/http` & RESTful API — Complete Offline Reference

> **Who this is for:** Go developers (beginner to intermediate) who want to build production-quality HTTP servers and REST APIs using **only the Go standard library** — no Gin, Echo, Chi, or Fiber required. Every concept is explained with runnable, commented code you can paste straight into a `main.go` and run with `go run .`.
>
> **Version note:** This guide targets **Go 1.22+**. The single biggest change in that release was a massively enhanced `net/http.ServeMux` that supports method-based routing, wildcard path segments, and catch-all patterns natively. Where behaviour differs from older Go versions it is flagged with **⚡ Version note**. Confirm exact APIs at pkg.go.dev/net/http.

---

## Table of Contents

1. [Why `net/http` — No Framework Needed](#1-why-nethttp--no-framework-needed)
2. [Hello Server — Foundational Types](#2-hello-server--foundational-types)
3. [ServeMux & Routing (Classic vs Go 1.22+)](#3-servemux--routing)
4. [Reading Requests](#4-reading-requests)
5. [Writing Responses](#5-writing-responses)
6. [Middleware Pattern in Go](#6-middleware-pattern-in-go)
7. [Full CRUD REST API — Worked Example](#7-full-crud-rest-api--worked-example)
8. [Context — Timeouts, Cancellation, Values](#8-context)
9. [Structured Logging with `log/slog`](#9-structured-logging-with-logslog)
10. [Graceful Shutdown](#10-graceful-shutdown)
11. [Serving Static Files](#11-serving-static-files)
12. [`http.Client` — Making Outbound Requests](#12-httpclient--making-outbound-requests)
13. [Testing Handlers with `net/http/httptest`](#13-testing-handlers-with-nethttphttptest)
14. [Idiomatic Project Structure](#14-idiomatic-project-structure)
15. [Tips, Tricks & Gotchas](#15-tips-tricks--gotchas)
16. [Study Path](#16-study-path)

---

## 1. Why `net/http` — No Framework Needed

### The standard library is production-grade

The Go standard library's `net/http` package ships a **fully concurrent, production-ready HTTP/1.1 and HTTP/2 server** with:

- A multiplexer (router) that now supports method and wildcard patterns (Go 1.22+)
- TLS via `ListenAndServeTLS`
- HTTP/2 automatically negotiated over TLS
- Keep-alive connection management
- Request and response body streaming
- A powerful `http.Client` for outbound requests
- Cookie, header, and multipart/form-data support
- A file server

You do *not* need a third-party framework to build a maintainable, performant API. Frameworks like Gin add convenience, but they also add a dependency, a learning curve, and upgrade churn. For most APIs — even large ones — the standard library is the right choice.

### What lives in the package

| Symbol | Role |
|---|---|
| `http.ListenAndServe` | Start an HTTP server |
| `http.ListenAndServeTLS` | Start an HTTPS server |
| `http.Server` | Configurable server struct (timeouts, TLS, etc.) |
| `http.ServeMux` | Request multiplexer / router |
| `http.Handler` | Interface: anything with `ServeHTTP(ResponseWriter, *Request)` |
| `http.HandlerFunc` | Adapter: converts a function to `http.Handler` |
| `http.ResponseWriter` | Interface for writing the HTTP response |
| `*http.Request` | Incoming request (URL, headers, body, context) |
| `http.Client` | Outbound HTTP client |
| `http.FileServer` | Serves files from disk |
| `http.StripPrefix` | Strips a URL prefix before passing to another handler |
| `http.Error` | Convenience: write an error response with a status code |
| `http.NotFound` | Write 404 |
| `http.Redirect` | Write 3xx redirect |
| `http.Cookie` / `http.SetCookie` | Cookie support |

---

## 2. Hello Server — Foundational Types

### The minimal server

```go
// main.go
package main

import (
    "fmt"
    "log"
    "net/http"
)

func helloHandler(w http.ResponseWriter, r *http.Request) {
    // w is the http.ResponseWriter — we write the response to it.
    // r is the *http.Request — it holds everything about the incoming request.
    fmt.Fprintln(w, "Hello, World!")
}

func main() {
    // Register helloHandler for every request to "/".
    // http.HandleFunc registers on the *default* package-level ServeMux.
    http.HandleFunc("/", helloHandler)

    // ListenAndServe blocks forever (or until it errors).
    // The second argument is the handler — nil means "use the default ServeMux".
    log.Println("Listening on :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

Run with:

```bash
go run main.go
curl http://localhost:8080/
# → Hello, World!
```

---

### The `http.Handler` interface

Everything in `net/http` revolves around this single interface:

```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```

Any type that implements `ServeHTTP` is a handler. This is the key abstraction — middleware, routers, and individual endpoints all implement (or adapt to) this interface.

---

### `http.HandlerFunc` — the function adapter

Writing a full struct for every simple handler is verbose. `http.HandlerFunc` lets you use a plain function:

```go
// HandlerFunc is defined in the standard library as:
// type HandlerFunc func(ResponseWriter, *Request)
// func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) { f(w, r) }

// So these two are equivalent:
mux.Handle("/ping", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, "pong")
}))

// The shortcut (mux.HandleFunc wraps it for you):
mux.HandleFunc("/ping", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, "pong")
})
```

---

### `http.ResponseWriter`

`ResponseWriter` is an interface with three methods:

```go
type ResponseWriter interface {
    // Header returns the response headers map. You must set headers BEFORE
    // calling WriteHeader or Write, because WriteHeader sends them.
    Header() http.Header

    // Write writes the body. If WriteHeader has not been called, it implicitly
    // calls WriteHeader(http.StatusOK) first.
    Write([]byte) (int, error)

    // WriteHeader sends the HTTP status code. Can only be called once.
    // After this call, Header() changes have no effect.
    WriteHeader(statusCode int)
}
```

Common pattern:

```go
func myHandler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated) // 201
    w.Write([]byte(`{"id": 1}`))
}
```

---

### `*http.Request` — key fields

```go
r.Method        // "GET", "POST", "PUT", "DELETE", etc.
r.URL           // *url.URL — r.URL.Path, r.URL.RawQuery, r.URL.Query()
r.Header        // http.Header (map[string][]string)
r.Body          // io.ReadCloser — always close it
r.Context()     // request context — for cancellation, deadlines, values
r.Form          // populated after r.ParseForm() or r.ParseMultipartForm()
r.PostForm      // form data from the body only (not URL params)
r.RemoteAddr    // client address "IP:port"
r.TLS           // non-nil if the connection is HTTPS
```

---

## 3. ServeMux & Routing

### Classic ServeMux (before Go 1.22)

Before Go 1.22 the built-in multiplexer was very basic — fixed strings and subtree patterns only:

```go
mux := http.NewServeMux()

// Exact match: only "/about"
mux.HandleFunc("/about", aboutHandler)

// Subtree match: "/static/" matches "/static/css/app.css" etc.
mux.HandleFunc("/static/", staticHandler)
```

No method routing, no path parameters. Third-party routers (chi, gorilla/mux) existed largely to fill this gap.

---

### Go 1.22+ Enhanced ServeMux

⚡ **Version note:** Go 1.22 (released February 2024) shipped a dramatically improved `ServeMux`. You get method routing, wildcard segments, and catch-all tails natively.

#### Method-based patterns

```go
mux := http.NewServeMux()

// Pattern: "METHOD /path"
// Only matches GET requests to /users
mux.HandleFunc("GET /users", listUsersHandler)

// Only matches POST requests to /users
mux.HandleFunc("POST /users", createUserHandler)

// Method + wildcard segment
mux.HandleFunc("GET /users/{id}", getUserHandler)
mux.HandleFunc("PUT /users/{id}", updateUserHandler)
mux.HandleFunc("DELETE /users/{id}", deleteUserHandler)
```

#### Wildcard segments `{name}`

```go
// {id} matches exactly ONE path segment (no slashes).
// Retrieve the value with r.PathValue("id").
mux.HandleFunc("GET /books/{id}", func(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")  // e.g. "42" for GET /books/42
    fmt.Fprintf(w, "Book ID: %s\n", id)
})
```

#### Catch-all (tail) wildcard `{path...}`

```go
// {path...} matches the rest of the URL including slashes.
// Useful for file servers and proxy-style handlers.
mux.HandleFunc("GET /files/{path...}", func(w http.ResponseWriter, r *http.Request) {
    rest := r.PathValue("path") // e.g. "images/logo.png" for /files/images/logo.png
    fmt.Fprintf(w, "Serving: %s\n", rest)
})
```

#### Routing precedence rules

When multiple patterns could match, Go 1.22+ uses **specificity** (not registration order):

| Priority | Wins when |
|---|---|
| 1 (highest) | Method + exact path (`GET /users/me`) |
| 2 | Method + wildcard path (`GET /users/{id}`) |
| 3 | No method + exact path (`/users/me`) |
| 4 | No method + wildcard path (`/users/{id}`) |
| 5 (lowest) | Subtree (`/`) |

More specific patterns beat less specific ones. If two patterns are equally specific and overlap, the router panics at startup — a deliberate design to catch ambiguous routes.

```go
// This is fine: "GET /users/me" is more specific than "GET /users/{id}"
mux.HandleFunc("GET /users/me", getMeHandler)      // matches only /users/me
mux.HandleFunc("GET /users/{id}", getUserHandler)  // matches /users/42, /users/abc, etc.
```

#### Full routing example

```go
package main

import (
    "fmt"
    "log"
    "net/http"
)

func main() {
    mux := http.NewServeMux()

    // Health check — no method prefix means any method
    mux.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintln(w, `{"status":"ok"}`)
    })

    // Users resource
    mux.HandleFunc("GET /users",        listUsers)
    mux.HandleFunc("POST /users",       createUser)
    mux.HandleFunc("GET /users/{id}",   getUser)
    mux.HandleFunc("PUT /users/{id}",   updateUser)
    mux.HandleFunc("DELETE /users/{id}", deleteUser)

    // Nested resource: posts belonging to a user
    mux.HandleFunc("GET /users/{userID}/posts/{postID}", getPost)

    log.Fatal(http.ListenAndServe(":8080", mux))
}

func listUsers(w http.ResponseWriter, r *http.Request)   { fmt.Fprintln(w, "list users") }
func createUser(w http.ResponseWriter, r *http.Request)  { fmt.Fprintln(w, "create user") }
func getUser(w http.ResponseWriter, r *http.Request)     { fmt.Fprintf(w, "get user %s\n", r.PathValue("id")) }
func updateUser(w http.ResponseWriter, r *http.Request)  { fmt.Fprintf(w, "update user %s\n", r.PathValue("id")) }
func deleteUser(w http.ResponseWriter, r *http.Request)  { fmt.Fprintf(w, "delete user %s\n", r.PathValue("id")) }

func getPost(w http.ResponseWriter, r *http.Request) {
    userID := r.PathValue("userID")
    postID := r.PathValue("postID")
    fmt.Fprintf(w, "user=%s post=%s\n", userID, postID)
}
```

---

### Using your own mux vs the default mux

```go
// Default mux (package-level): fine for tiny programs, avoid in production.
http.HandleFunc("/", handler)
http.ListenAndServe(":8080", nil) // nil → uses default mux

// Explicit mux: always prefer this. Testable, no global state.
mux := http.NewServeMux()
mux.HandleFunc("/", handler)
http.ListenAndServe(":8080", mux)
```

> **Gotcha:** Third-party packages that call `http.HandleFunc` at init time pollute the default mux. Always use an explicit `http.NewServeMux()`.

---

## 4. Reading Requests

### Path parameters with `r.PathValue`

⚡ **Version note:** `r.PathValue` is Go 1.22+. On older Go you need a third-party router.

```go
mux.HandleFunc("GET /products/{category}/{id}", func(w http.ResponseWriter, r *http.Request) {
    category := r.PathValue("category") // e.g. "electronics"
    id       := r.PathValue("id")       // e.g. "99"

    // PathValue always returns a string. Parse it as needed:
    productID, err := strconv.Atoi(id)
    if err != nil {
        http.Error(w, "invalid id", http.StatusBadRequest)
        return
    }

    fmt.Fprintf(w, "category=%s id=%d\n", category, productID)
})
```

---

### Query parameters

```go
// URL: /search?q=golang&page=2&limit=20
mux.HandleFunc("GET /search", func(w http.ResponseWriter, r *http.Request) {
    // r.URL.Query() returns url.Values (map[string][]string), never an error.
    q     := r.URL.Query()
    query := q.Get("q")          // "golang"    (empty string if missing)
    page  := q.Get("page")       // "2"
    limit := q.Get("limit")      // "20"

    // Multiple values for the same key:
    tags := q["tag"] // []string{"go","web"} for ?tag=go&tag=web

    fmt.Fprintf(w, "q=%s page=%s limit=%s tags=%v\n", query, page, limit, tags)
})
```

---

### Request headers

```go
func myHandler(w http.ResponseWriter, r *http.Request) {
    // Header names are canonicalised — case-insensitive via textproto.CanonicalMIMEHeaderKey.
    auth        := r.Header.Get("Authorization")       // "Bearer eyJ..."
    contentType := r.Header.Get("Content-Type")        // "application/json"
    accept      := r.Header.Get("Accept")

    // Multiple values:
    values := r.Header["Accept-Encoding"] // []string

    _ = auth; _ = contentType; _ = accept; _ = values
}
```

---

### JSON body decoding

```go
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
)

type CreateUserRequest struct {
    Name  string `json:"name"`
    Email string `json:"email"`
    Age   int    `json:"age"`
}

func createUserHandler(w http.ResponseWriter, r *http.Request) {
    // Limit body size to prevent abuse (1 MB here).
    r.Body = http.MaxBytesReader(w, r.Body, 1<<20)
    defer r.Body.Close() // ALWAYS close the body

    var req CreateUserRequest

    // json.NewDecoder streams from the body — more memory-efficient than ioutil.ReadAll + Unmarshal.
    dec := json.NewDecoder(r.Body)
    dec.DisallowUnknownFields() // return an error if the client sends unexpected fields

    if err := dec.Decode(&req); err != nil {
        http.Error(w, "invalid JSON: "+err.Error(), http.StatusBadRequest)
        return
    }

    // Validation
    if req.Name == "" {
        http.Error(w, "name is required", http.StatusUnprocessableEntity)
        return
    }

    fmt.Fprintf(w, "created user: %+v\n", req)
}
```

---

### Form data (HTML forms, URL-encoded)

```go
func formHandler(w http.ResponseWriter, r *http.Request) {
    // ParseForm populates r.Form (URL params + body) and r.PostForm (body only).
    if err := r.ParseForm(); err != nil {
        http.Error(w, "bad form", http.StatusBadRequest)
        return
    }

    name  := r.FormValue("name")  // shortcut: calls ParseForm if needed
    email := r.PostFormValue("email") // body only, ignores URL query

    fmt.Fprintf(w, "name=%s email=%s\n", name, email)
}
```

---

### Multipart / file uploads

```go
func uploadHandler(w http.ResponseWriter, r *http.Request) {
    // 32 MB max memory; remainder spills to temp files.
    if err := r.ParseMultipartForm(32 << 20); err != nil {
        http.Error(w, "bad multipart", http.StatusBadRequest)
        return
    }

    file, header, err := r.FormFile("avatar")
    if err != nil {
        http.Error(w, "no file", http.StatusBadRequest)
        return
    }
    defer file.Close()

    fmt.Fprintf(w, "uploaded: %s (%d bytes)\n", header.Filename, header.Size)
    // Read file with io.Copy(dst, file)
}
```

---

## 5. Writing Responses

### Status codes

Use the `http.Status*` constants — never magic numbers:

```go
http.StatusOK                   // 200
http.StatusCreated              // 201
http.StatusNoContent            // 204
http.StatusBadRequest           // 400
http.StatusUnauthorized         // 401
http.StatusForbidden            // 403
http.StatusNotFound             // 404
http.StatusMethodNotAllowed     // 405
http.StatusConflict             // 409
http.StatusUnprocessableEntity  // 422
http.StatusInternalServerError  // 500
```

---

### Setting headers and writing status

```go
func handler(w http.ResponseWriter, r *http.Request) {
    // Set headers FIRST — before WriteHeader or Write.
    w.Header().Set("Content-Type", "application/json")
    w.Header().Set("X-Request-Id", "abc123")

    // Then set status code.
    w.WriteHeader(http.StatusCreated) // 201

    // Then write body.
    w.Write([]byte(`{"id":1,"name":"Alice"}`))
}
```

---

### JSON encoding helper

Write a reusable helper and use it everywhere:

```go
package main

import (
    "encoding/json"
    "net/http"
)

// writeJSON encodes v as JSON and writes it with the given status code.
func writeJSON(w http.ResponseWriter, status int, v any) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    if err := json.NewEncoder(w).Encode(v); err != nil {
        // At this point headers are already sent, so log and move on.
        // Don't call http.Error here — it's too late.
        _ = err
    }
}

// writeError writes a JSON error response.
func writeError(w http.ResponseWriter, status int, message string) {
    writeJSON(w, status, map[string]string{"error": message})
}

// Usage:
func getItemHandler(w http.ResponseWriter, r *http.Request) {
    item := map[string]any{"id": 1, "name": "widget"}
    writeJSON(w, http.StatusOK, item)
}

func notFoundHandler(w http.ResponseWriter, r *http.Request) {
    writeError(w, http.StatusNotFound, "item not found")
}
```

---

### Consistent error response type

```go
// ErrorResponse is the shape every error returns.
type ErrorResponse struct {
    Error  string `json:"error"`
    Detail string `json:"detail,omitempty"`
    Code   int    `json:"code"`
}

func apiError(w http.ResponseWriter, status int, msg string) {
    resp := ErrorResponse{
        Error: http.StatusText(status),
        Detail: msg,
        Code:  status,
    }
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(resp)
}
```

---

## 6. Middleware Pattern in Go

### What is middleware?

Middleware is a function that wraps an `http.Handler` to add behaviour (logging, auth, CORS, panic recovery, etc.) before or after the wrapped handler runs.

The signature is always:

```go
func(next http.Handler) http.Handler
```

### Logging middleware

```go
package main

import (
    "log"
    "net/http"
    "time"
)

// Logger wraps next and logs each request's method, path, and duration.
func Logger(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()

        // Call the actual handler.
        next.ServeHTTP(w, r)

        log.Printf("%s %s %s", r.Method, r.URL.Path, time.Since(start))
    })
}
```

### Auth middleware

```go
// RequireAuth checks the Authorization header for a valid Bearer token.
func RequireAuth(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")
        if token != "Bearer secret-token" {
            http.Error(w, "unauthorized", http.StatusUnauthorized)
            return // stop the chain — do NOT call next
        }
        next.ServeHTTP(w, r) // token valid, continue
    })
}
```

### Panic recovery middleware

```go
import (
    "log"
    "net/http"
    "runtime/debug"
)

// Recoverer catches panics in handlers and returns 500 instead of crashing.
func Recoverer(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                log.Printf("panic: %v\n%s", err, debug.Stack())
                http.Error(w, "internal server error", http.StatusInternalServerError)
            }
        }()
        next.ServeHTTP(w, r)
    })
}
```

### CORS middleware

```go
// CORS adds permissive CORS headers. Tighten origins for production.
func CORS(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Access-Control-Allow-Origin", "*")
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
        w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")

        // Handle preflight requests.
        if r.Method == http.MethodOptions {
            w.WriteHeader(http.StatusNoContent)
            return
        }

        next.ServeHTTP(w, r)
    })
}
```

### Chaining middleware

Go has no built-in middleware chain helper, but the pattern is simple:

```go
func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("GET /users", listUsersHandler)

    // Apply middleware right-to-left (innermost is closest to the handler):
    // Request flow: Logger → Recoverer → CORS → mux handler
    handler := Logger(Recoverer(CORS(mux)))

    http.ListenAndServe(":8080", handler)
}
```

Or write a tiny helper:

```go
// Chain applies middleware in left-to-right order.
// chain(Logger, Auth, CORS)(mux) → Logger → Auth → CORS → mux
func Chain(h http.Handler, middleware ...func(http.Handler) http.Handler) http.Handler {
    // Apply in reverse so the first listed is the outermost.
    for i := len(middleware) - 1; i >= 0; i-- {
        h = middleware[i](h)
    }
    return h
}

// Usage:
handler := Chain(mux, Logger, Recoverer, CORS)
```

### Per-route middleware

You can apply middleware to individual routes too:

```go
mux.Handle("GET /admin/users", RequireAuth(http.HandlerFunc(adminListUsers)))
```

---

## 7. Full CRUD REST API — Worked Example

This section builds a complete **Books API** with all CRUD endpoints, a handler struct for dependency injection, proper error handling, and JSON request/response types — in a single file you can run immediately.

### Data types and storage

```go
package main

import (
    "encoding/json"
    "errors"
    "fmt"
    "log"
    "net/http"
    "strconv"
    "sync"
    "time"
)

// Book is the domain model.
type Book struct {
    ID        int       `json:"id"`
    Title     string    `json:"title"`
    Author    string    `json:"author"`
    Year      int       `json:"year"`
    CreatedAt time.Time `json:"created_at"`
}

// CreateBookRequest is the shape of POST /books body.
type CreateBookRequest struct {
    Title  string `json:"title"`
    Author string `json:"author"`
    Year   int    `json:"year"`
}

// UpdateBookRequest is the shape of PUT /books/{id} body.
type UpdateBookRequest struct {
    Title  string `json:"title,omitempty"`
    Author string `json:"author,omitempty"`
    Year   int    `json:"year,omitempty"`
}

// Store is an in-memory thread-safe book store.
type Store struct {
    mu     sync.RWMutex
    books  map[int]Book
    nextID int
}

func NewStore() *Store {
    return &Store{books: make(map[int]Book), nextID: 1}
}

var ErrNotFound = errors.New("not found")

func (s *Store) List() []Book {
    s.mu.RLock()
    defer s.mu.RUnlock()
    out := make([]Book, 0, len(s.books))
    for _, b := range s.books {
        out = append(out, b)
    }
    return out
}

func (s *Store) Get(id int) (Book, error) {
    s.mu.RLock()
    defer s.mu.RUnlock()
    b, ok := s.books[id]
    if !ok {
        return Book{}, ErrNotFound
    }
    return b, nil
}

func (s *Store) Create(req CreateBookRequest) Book {
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
    return b
}

func (s *Store) Update(id int, req UpdateBookRequest) (Book, error) {
    s.mu.Lock()
    defer s.mu.Unlock()
    b, ok := s.books[id]
    if !ok {
        return Book{}, ErrNotFound
    }
    if req.Title != "" {
        b.Title = req.Title
    }
    if req.Author != "" {
        b.Author = req.Author
    }
    if req.Year != 0 {
        b.Year = req.Year
    }
    s.books[id] = b
    return b, nil
}

func (s *Store) Delete(id int) error {
    s.mu.Lock()
    defer s.mu.Unlock()
    if _, ok := s.books[id]; !ok {
        return ErrNotFound
    }
    delete(s.books, id)
    return nil
}
```

### The handler struct

```go
// BooksHandler holds dependencies and implements all book endpoints.
// This pattern (handler struct) is idiomatic Go for APIs — it avoids globals
// and makes handlers trivially testable.
type BooksHandler struct {
    store *Store
}

func NewBooksHandler(store *Store) *BooksHandler {
    return &BooksHandler{store: store}
}
```

### Helper functions

```go
func writeJSON(w http.ResponseWriter, status int, v any) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(v)
}

func writeError(w http.ResponseWriter, status int, msg string) {
    writeJSON(w, status, map[string]string{"error": msg})
}

func parseID(r *http.Request) (int, error) {
    return strconv.Atoi(r.PathValue("id"))
}
```

### Endpoints

```go
// List: GET /books
func (h *BooksHandler) List(w http.ResponseWriter, r *http.Request) {
    books := h.store.List()
    writeJSON(w, http.StatusOK, map[string]any{
        "data":  books,
        "count": len(books),
    })
}

// Get: GET /books/{id}
func (h *BooksHandler) Get(w http.ResponseWriter, r *http.Request) {
    id, err := parseID(r)
    if err != nil {
        writeError(w, http.StatusBadRequest, "id must be an integer")
        return
    }

    book, err := h.store.Get(id)
    if errors.Is(err, ErrNotFound) {
        writeError(w, http.StatusNotFound, fmt.Sprintf("book %d not found", id))
        return
    }

    writeJSON(w, http.StatusOK, book)
}

// Create: POST /books
func (h *BooksHandler) Create(w http.ResponseWriter, r *http.Request) {
    r.Body = http.MaxBytesReader(w, r.Body, 1<<20)
    defer r.Body.Close()

    var req CreateBookRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        writeError(w, http.StatusBadRequest, "invalid JSON: "+err.Error())
        return
    }

    // Validation
    if req.Title == "" {
        writeError(w, http.StatusUnprocessableEntity, "title is required")
        return
    }
    if req.Author == "" {
        writeError(w, http.StatusUnprocessableEntity, "author is required")
        return
    }

    book := h.store.Create(req)
    writeJSON(w, http.StatusCreated, book)
}

// Update: PUT /books/{id}
func (h *BooksHandler) Update(w http.ResponseWriter, r *http.Request) {
    id, err := parseID(r)
    if err != nil {
        writeError(w, http.StatusBadRequest, "id must be an integer")
        return
    }

    r.Body = http.MaxBytesReader(w, r.Body, 1<<20)
    defer r.Body.Close()

    var req UpdateBookRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        writeError(w, http.StatusBadRequest, "invalid JSON: "+err.Error())
        return
    }

    book, err := h.store.Update(id, req)
    if errors.Is(err, ErrNotFound) {
        writeError(w, http.StatusNotFound, fmt.Sprintf("book %d not found", id))
        return
    }

    writeJSON(w, http.StatusOK, book)
}

// Delete: DELETE /books/{id}
func (h *BooksHandler) Delete(w http.ResponseWriter, r *http.Request) {
    id, err := parseID(r)
    if err != nil {
        writeError(w, http.StatusBadRequest, "id must be an integer")
        return
    }

    if err := h.store.Delete(id); errors.Is(err, ErrNotFound) {
        writeError(w, http.StatusNotFound, fmt.Sprintf("book %d not found", id))
        return
    }

    w.WriteHeader(http.StatusNoContent) // 204 — no body on successful delete
}
```

### Wiring it together

```go
func main() {
    store := NewStore()
    h := NewBooksHandler(store)

    mux := http.NewServeMux()

    // Books resource
    mux.HandleFunc("GET /books",         h.List)
    mux.HandleFunc("POST /books",        h.Create)
    mux.HandleFunc("GET /books/{id}",    h.Get)
    mux.HandleFunc("PUT /books/{id}",    h.Update)
    mux.HandleFunc("DELETE /books/{id}", h.Delete)

    // Health
    mux.HandleFunc("GET /health", func(w http.ResponseWriter, r *http.Request) {
        writeJSON(w, http.StatusOK, map[string]string{"status": "ok"})
    })

    // Apply global middleware
    handler := Logger(Recoverer(mux))

    srv := &http.Server{
        Addr:         ":8080",
        Handler:      handler,
        ReadTimeout:  5 * time.Second,
        WriteTimeout: 10 * time.Second,
        IdleTimeout:  120 * time.Second,
    }

    log.Println("Books API listening on :8080")
    log.Fatal(srv.ListenAndServe())
}
```

### Testing with curl

```bash
# Create
curl -X POST http://localhost:8080/books \
  -H "Content-Type: application/json" \
  -d '{"title":"The Go Programming Language","author":"Donovan & Kernighan","year":2015}'

# List
curl http://localhost:8080/books

# Get one
curl http://localhost:8080/books/1

# Update
curl -X PUT http://localhost:8080/books/1 \
  -H "Content-Type: application/json" \
  -d '{"year":2016}'

# Delete
curl -X DELETE http://localhost:8080/books/1
```

---

## 8. Context

### What is context?

`context.Context` is the standard Go mechanism for:
- **Cancellation** — tell downstream work to stop (request cancelled, client disconnected)
- **Deadlines / timeouts** — automatically cancel after a duration
- **Values** — pass request-scoped data (user ID, trace ID) without function arguments

Every `*http.Request` carries a context from the moment it arrives. When the client disconnects, the context is cancelled automatically.

### Using request context

```go
import (
    "context"
    "database/sql"
    "net/http"
)

func getUserHandler(w http.ResponseWriter, r *http.Request) {
    // r.Context() is the request's context.
    // Pass it to any function that does I/O so it can be cancelled.
    ctx := r.Context()

    id := r.PathValue("id")

    // Hypothetical DB call — if the client disconnects mid-query, ctx is cancelled
    // and the DB driver will abort the query.
    row := db.QueryRowContext(ctx, "SELECT name FROM users WHERE id = $1", id)
    // ...
}
```

### Adding a timeout to the request context

```go
func slowHandler(w http.ResponseWriter, r *http.Request) {
    // Add a 2-second deadline on top of whatever the request context already has.
    ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
    defer cancel() // ALWAYS defer cancel to avoid context leak

    result, err := doSlowWork(ctx)
    if err != nil {
        if ctx.Err() != nil {
            // Context expired or was cancelled.
            http.Error(w, "request timed out", http.StatusGatewayTimeout)
            return
        }
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    writeJSON(w, http.StatusOK, result)
}
```

### Passing values through context

Use typed keys to avoid collisions:

```go
// Define a private key type — prevents other packages from clashing.
type contextKey string

const keyUserID contextKey = "userID"

// SetUserID middleware extracts a user from the token and stores it in context.
func SetUserID(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        userID := extractUserIDFromToken(r.Header.Get("Authorization"))
        ctx := context.WithValue(r.Context(), keyUserID, userID)
        next.ServeHTTP(w, r.WithContext(ctx)) // r.WithContext returns a shallow copy
    })
}

// GetUserID retrieves the user ID stored by SetUserID middleware.
func GetUserID(ctx context.Context) (string, bool) {
    id, ok := ctx.Value(keyUserID).(string)
    return id, ok
}

// In a handler:
func meHandler(w http.ResponseWriter, r *http.Request) {
    userID, ok := GetUserID(r.Context())
    if !ok {
        http.Error(w, "not authenticated", http.StatusUnauthorized)
        return
    }
    fmt.Fprintf(w, "hello user %s\n", userID)
}
```

### Context cancellation — checking if the client left

```go
func streamHandler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()

    for i := 0; i < 100; i++ {
        select {
        case <-ctx.Done():
            // Client disconnected. Stop work, don't write any more.
            log.Println("client disconnected:", ctx.Err())
            return
        default:
            fmt.Fprintf(w, "chunk %d\n", i)
            if f, ok := w.(http.Flusher); ok {
                f.Flush() // push data to the client immediately
            }
            time.Sleep(100 * time.Millisecond)
        }
    }
}
```

---

## 9. Structured Logging with `log/slog`

⚡ **Version note:** `log/slog` was added in **Go 1.21**. It provides structured, levelled logging with JSON output out of the box — no third-party library needed.

### Basic slog usage

```go
package main

import (
    "log/slog"
    "net/http"
    "os"
)

func main() {
    // Create a JSON logger writing to stdout.
    logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelDebug, // log everything at Debug and above
    }))

    // Set as the default so slog.Info etc. use it.
    slog.SetDefault(logger)

    slog.Info("server starting", "addr", ":8080")

    mux := http.NewServeMux()
    mux.HandleFunc("GET /ping", func(w http.ResponseWriter, r *http.Request) {
        slog.Info("ping received", "remote", r.RemoteAddr)
        w.Write([]byte("pong"))
    })

    http.ListenAndServe(":8080", mux)
}
```

Output:
```json
{"time":"2026-06-19T10:00:00Z","level":"INFO","msg":"server starting","addr":":8080"}
{"time":"2026-06-19T10:00:01Z","level":"INFO","msg":"ping received","remote":"127.0.0.1:54321"}
```

### Structured logging middleware

```go
import (
    "log/slog"
    "net/http"
    "time"
)

// SlogLogger is a middleware that logs each request using slog.
func SlogLogger(logger *slog.Logger) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            start := time.Now()

            // Wrap ResponseWriter to capture the status code.
            rw := &responseWriter{ResponseWriter: w, status: http.StatusOK}
            next.ServeHTTP(rw, r)

            logger.Info("request",
                "method",   r.Method,
                "path",     r.URL.Path,
                "status",   rw.status,
                "duration", time.Since(start).String(),
                "remote",   r.RemoteAddr,
            )
        })
    }
}

// responseWriter wraps http.ResponseWriter to capture the written status code.
type responseWriter struct {
    http.ResponseWriter
    status int
}

func (rw *responseWriter) WriteHeader(code int) {
    rw.status = code
    rw.ResponseWriter.WriteHeader(code)
}
```

### Logger available in handlers via context

```go
// Pass the logger through context so handlers can use it without global state.
const keyLogger contextKey = "logger"

func WithLogger(logger *slog.Logger) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            ctx := context.WithValue(r.Context(), keyLogger, logger)
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}

func LoggerFromCtx(ctx context.Context) *slog.Logger {
    if l, ok := ctx.Value(keyLogger).(*slog.Logger); ok {
        return l
    }
    return slog.Default()
}
```

---

## 10. Graceful Shutdown

A server that exits abruptly drops in-flight requests. Graceful shutdown waits for them to finish.

```go
package main

import (
    "context"
    "log/slog"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"
)

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("GET /", func(w http.ResponseWriter, r *http.Request) {
        // Simulate a slow request.
        time.Sleep(3 * time.Second)
        w.Write([]byte("done"))
    })

    srv := &http.Server{
        Addr:         ":8080",
        Handler:      mux,
        ReadTimeout:  5 * time.Second,
        WriteTimeout: 10 * time.Second,
        IdleTimeout:  120 * time.Second,
    }

    // Start the server in a goroutine so it doesn't block.
    go func() {
        slog.Info("server started", "addr", srv.Addr)
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            slog.Error("listen error", "err", err)
            os.Exit(1)
        }
    }()

    // ⚡ Version note: signal.NotifyContext is available since Go 1.16.
    // It creates a context that is cancelled when SIGINT or SIGTERM is received.
    ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
    defer stop()

    // Block until a signal is received.
    <-ctx.Done()
    stop() // reset signal handling so a second Ctrl+C kills immediately

    slog.Info("shutting down...")

    // Give in-flight requests up to 10 seconds to finish.
    shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    if err := srv.Shutdown(shutdownCtx); err != nil {
        slog.Error("shutdown error", "err", err)
        os.Exit(1)
    }

    slog.Info("shutdown complete")
}
```

When you press Ctrl+C:
1. `ctx.Done()` unblocks
2. `srv.Shutdown` stops accepting new connections
3. It waits for active handlers to finish (up to 10 s)
4. Process exits cleanly

---

## 11. Serving Static Files

### Basic file server

```go
mux := http.NewServeMux()

// Serve everything under ./public at /static/
// http.StripPrefix removes "/static" before passing to the file server.
fs := http.FileServer(http.Dir("./public"))
mux.Handle("/static/", http.StripPrefix("/static/", fs))
```

Access `./public/css/app.css` at `http://localhost:8080/static/css/app.css`.

### Embedding files with `//go:embed`

⚡ **Version note:** `embed` package available since Go 1.16. Ship static files inside the binary — no separate `public/` folder needed in production.

```go
package main

import (
    "embed"
    "net/http"
)

//go:embed public
var publicFS embed.FS

func main() {
    mux := http.NewServeMux()

    // http.FS wraps embed.FS so it satisfies http.FileSystem.
    mux.Handle("/static/", http.StripPrefix("/static/", http.FileServer(http.FS(publicFS))))

    http.ListenAndServe(":8080", mux)
}
```

### SPA catch-all (serve index.html for every route)

```go
func spaHandler(publicDir string) http.HandlerFunc {
    fs := http.FileServer(http.Dir(publicDir))
    return func(w http.ResponseWriter, r *http.Request) {
        // Try to serve the real file. If it doesn't exist, serve index.html.
        path := publicDir + r.URL.Path
        if _, err := os.Stat(path); os.IsNotExist(err) {
            http.ServeFile(w, r, publicDir+"/index.html")
            return
        }
        fs.ServeHTTP(w, r)
    }
}

mux.Handle("/", spaHandler("./dist"))
```

---

## 12. `http.Client` — Making Outbound Requests

### Always use a custom client with timeouts

```go
import (
    "encoding/json"
    "net/http"
    "time"
)

// Never use http.DefaultClient in production — it has no timeout.
var client = &http.Client{
    Timeout: 10 * time.Second, // total request time including reading body
}
```

### GET request

```go
func fetchUser(id string) (map[string]any, error) {
    resp, err := client.Get("https://api.example.com/users/" + id)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close() // ALWAYS close the body, even if you don't read it

    if resp.StatusCode != http.StatusOK {
        return nil, fmt.Errorf("upstream returned %d", resp.StatusCode)
    }

    var result map[string]any
    if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
        return nil, err
    }
    return result, nil
}
```

### POST request with JSON body

```go
import (
    "bytes"
    "encoding/json"
    "net/http"
)

func postJSON(url string, payload any) (*http.Response, error) {
    body, err := json.Marshal(payload)
    if err != nil {
        return nil, err
    }

    req, err := http.NewRequestWithContext(
        context.Background(),
        http.MethodPost,
        url,
        bytes.NewReader(body),
    )
    if err != nil {
        return nil, err
    }
    req.Header.Set("Content-Type", "application/json")

    return client.Do(req)
}
```

### Request with context (for cancellation)

```go
func fetchWithCtx(ctx context.Context, url string) ([]byte, error) {
    req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
    if err != nil {
        return nil, err
    }

    resp, err := client.Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    return io.ReadAll(resp.Body)
}
```

---

## 13. Testing Handlers with `net/http/httptest`

The `net/http/httptest` package lets you test handlers without starting a real TCP server.

### Testing a single handler

```go
package main

import (
    "encoding/json"
    "net/http"
    "net/http/httptest"
    "strings"
    "testing"
)

func TestCreateBook(t *testing.T) {
    // Set up handler
    store := NewStore()
    h := NewBooksHandler(store)

    // Build a fake request
    body := `{"title":"Test Book","author":"Test Author","year":2024}`
    req := httptest.NewRequest(http.MethodPost, "/books", strings.NewReader(body))
    req.Header.Set("Content-Type", "application/json")

    // Record the response
    w := httptest.NewRecorder()

    // Call the handler directly — no server needed
    h.Create(w, req)

    resp := w.Result()
    defer resp.Body.Close()

    // Assert status
    if resp.StatusCode != http.StatusCreated {
        t.Errorf("want 201, got %d", resp.StatusCode)
    }

    // Assert body
    var book Book
    if err := json.NewDecoder(resp.Body).Decode(&book); err != nil {
        t.Fatal("decode:", err)
    }
    if book.Title != "Test Book" {
        t.Errorf("want title 'Test Book', got %q", book.Title)
    }
    if book.ID == 0 {
        t.Error("expected ID to be set")
    }
}
```

### Testing with path values (Go 1.22+)

```go
func TestGetBook(t *testing.T) {
    store := NewStore()
    // Pre-populate
    store.Create(CreateBookRequest{Title: "Go in Action", Author: "Kennedy", Year: 2015})

    h := NewBooksHandler(store)

    // For path values, register on a test mux so ServeMux sets r.PathValue.
    mux := http.NewServeMux()
    mux.HandleFunc("GET /books/{id}", h.Get)

    req := httptest.NewRequest(http.MethodGet, "/books/1", nil)
    w := httptest.NewRecorder()

    mux.ServeHTTP(w, req) // use the mux, not the handler directly

    if w.Code != http.StatusOK {
        t.Errorf("want 200, got %d", w.Code)
    }
}
```

### Testing a 404

```go
func TestGetBookNotFound(t *testing.T) {
    store := NewStore()
    h := NewBooksHandler(store)

    mux := http.NewServeMux()
    mux.HandleFunc("GET /books/{id}", h.Get)

    req := httptest.NewRequest(http.MethodGet, "/books/999", nil)
    w := httptest.NewRecorder()
    mux.ServeHTTP(w, req)

    if w.Code != http.StatusNotFound {
        t.Errorf("want 404, got %d", w.Code)
    }
}
```

### Integration test with a real server

```go
func TestServer(t *testing.T) {
    store := NewStore()
    h := NewBooksHandler(store)
    mux := http.NewServeMux()
    mux.HandleFunc("GET /books", h.List)

    // httptest.NewServer starts a local server on an OS-assigned port.
    srv := httptest.NewServer(mux)
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

Run tests:

```bash
go test ./...         # all packages
go test -v ./...      # verbose — print each test name
go test -run TestGetBook ./...  # run matching tests only
go test -cover ./...  # show coverage
```

---

## 14. Idiomatic Project Structure

For a small-to-medium API, keep it flat and avoid over-engineering. Grow into structure as the project grows.

```
myapi/
├── cmd/
│   └── api/
│       └── main.go         # entry point: wires everything together
├── internal/
│   ├── handler/
│   │   ├── books.go        # BooksHandler and its methods
│   │   ├── health.go       # HealthHandler
│   │   └── middleware.go   # Logger, Recoverer, CORS, Auth
│   ├── store/
│   │   └── books.go        # Store (in-memory or database)
│   └── model/
│       └── book.go         # Book, CreateBookRequest, UpdateBookRequest
├── go.mod
└── go.sum
```

### `cmd/api/main.go` — the wiring file

```go
package main

import (
    "context"
    "log/slog"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"

    "myapi/internal/handler"
    "myapi/internal/store"
)

func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
    slog.SetDefault(logger)

    bookStore := store.NewBookStore()
    booksHandler := handler.NewBooksHandler(bookStore, logger)

    mux := http.NewServeMux()

    // Routes
    mux.HandleFunc("GET /health", handler.Health)
    mux.HandleFunc("GET /books",         booksHandler.List)
    mux.HandleFunc("POST /books",        booksHandler.Create)
    mux.HandleFunc("GET /books/{id}",    booksHandler.Get)
    mux.HandleFunc("PUT /books/{id}",    booksHandler.Update)
    mux.HandleFunc("DELETE /books/{id}", booksHandler.Delete)

    // Middleware chain
    h := handler.Chain(mux,
        handler.SlogLogger(logger),
        handler.Recoverer,
        handler.CORS,
    )

    addr := os.Getenv("PORT")
    if addr == "" {
        addr = ":8080"
    }

    srv := &http.Server{
        Addr:         addr,
        Handler:      h,
        ReadTimeout:  5 * time.Second,
        WriteTimeout: 10 * time.Second,
        IdleTimeout:  120 * time.Second,
    }

    go func() {
        logger.Info("server started", "addr", srv.Addr)
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            logger.Error("server error", "err", err)
            os.Exit(1)
        }
    }()

    ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
    defer stop()
    <-ctx.Done()

    shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()
    srv.Shutdown(shutdownCtx)
    logger.Info("shutdown complete")
}
```

### Rules of thumb for structure

| Rule | Why |
|---|---|
| `cmd/api/` for the main package | Multiple binaries live in separate `cmd/` subdirectories |
| `internal/` for app code | Prevents other modules from importing your internals |
| Handler struct per resource | Easy to test, injects dependencies cleanly |
| Keep `main.go` thin | Just wiring — no business logic |
| One file per responsibility | `handler/books.go` only has book handlers |
| Avoid `util/` and `helpers/` packages | They attract unrelated code; name packages by what they *do* |

---

## 15. Tips, Tricks & Gotchas

### 1. Always close `r.Body`

```go
// WRONG — body is never closed; connection can't be reused
func badHandler(w http.ResponseWriter, r *http.Request) {
    json.NewDecoder(r.Body).Decode(&v)
}

// RIGHT
func goodHandler(w http.ResponseWriter, r *http.Request) {
    defer r.Body.Close()
    json.NewDecoder(r.Body).Decode(&v)
}
```

### 2. Always close `http.Client` response bodies

```go
resp, _ := client.Get(url)
defer resp.Body.Close()      // even if you don't read the body
io.Copy(io.Discard, resp.Body) // drain it so the connection can be reused
```

### 3. Never use `http.DefaultClient` for production outbound calls

`http.DefaultClient` has no timeout. A slow upstream will block your goroutine forever.

```go
// BAD
resp, err := http.Get(url) // uses http.DefaultClient — no timeout

// GOOD
client := &http.Client{Timeout: 10 * time.Second}
resp, err := client.Get(url)
```

### 4. Always set server timeouts

Without timeouts, a slow or malicious client can hold connections open indefinitely:

```go
srv := &http.Server{
    Addr:         ":8080",
    Handler:      mux,
    ReadTimeout:  5 * time.Second,   // time to read headers + body
    WriteTimeout: 10 * time.Second,  // time to write the response
    IdleTimeout:  120 * time.Second, // keep-alive connection idle time
}
```

### 5. Avoid the default mux in packages

Calling `http.HandleFunc` in an `init()` or package-level function pollutes the global mux. Any library that does this is poorly behaved. Always use an explicit `http.NewServeMux()`.

### 6. `WriteHeader` can only be called once

Calling `w.WriteHeader` a second time does nothing (and logs a warning). Set all headers *before* the first `WriteHeader` or `Write` call.

```go
// WRONG — header won't be sent
func badHandler(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("hello"))         // implicitly sends 200
    w.Header().Set("X-Foo", "bar")  // too late — headers already sent
    w.WriteHeader(http.StatusCreated) // ignored
}
```

### 7. JSON `Encode` adds a trailing newline

`json.NewEncoder(w).Encode(v)` always appends `\n`. This is fine for APIs but can surprise you in tests comparing raw bytes. `json.Marshal` does not add a newline.

### 8. `r.PathValue` returns an empty string on no match

```go
id := r.PathValue("id") // "" if the pattern has no {id} wildcard
```

Always validate. Don't assume it's non-empty.

### 9. Method-pattern matching is case-sensitive for HTTP methods

`"get /users"` is not the same as `"GET /users"`. HTTP methods are uppercase by spec; always use uppercase in patterns.

### 10. Trailing slash in ServeMux patterns

```go
// "GET /api/v1/" with trailing slash = subtree match (matches all paths under /api/v1/)
// "GET /api/v1"  without = exact match only

mux.HandleFunc("GET /api/v1/", apiV1Handler)  // matches /api/v1/anything
mux.HandleFunc("GET /api/v1", apiV1Handler)   // only matches /api/v1 exactly
```

### 11. Panic in a handler crashes only that goroutine

The server stays up, but the connection is abruptly closed. Use a `Recoverer` middleware (section 6) in production.

### 12. JSON struct tags

```go
type User struct {
    ID        int    `json:"id"`
    FirstName string `json:"first_name"`            // snake_case for APIs
    Password  string `json:"-"`                     // never serialised
    CreatedAt string `json:"created_at,omitempty"`  // omit if zero value
}
```

### 13. `http.Error` sets `Content-Type: text/plain` and a newline

```go
http.Error(w, "not found", http.StatusNotFound)
// Sends: "not found\n" as text/plain
// Fine for simple errors, but use writeJSON for consistent API responses
```

### 14. The ServeMux automatically redirects `/path` to `/path/` for subtree patterns

If you register `/api/` and a request comes in for `/api`, the mux sends a 301 redirect to `/api/`. If that's not what you want, register both patterns.

### 15. Check `ctx.Err()` after operations that take a context

```go
result, err := db.QueryContext(ctx, query)
if err != nil {
    if ctx.Err() != nil {
        // The context was cancelled or timed out — not a DB error
        http.Error(w, "request cancelled", http.StatusRequestTimeout)
        return
    }
    http.Error(w, err.Error(), http.StatusInternalServerError)
    return
}
```

---

## 16. Study Path

Work through these in order. Each step builds on the last.

### Phase 1 — Core mechanics (1–2 days)

1. **Hello server** — `http.HandleFunc`, `http.ListenAndServe`, test with curl
2. **Routing** — Create an explicit `ServeMux`, add method-based routes, test `r.PathValue`
3. **Reading requests** — Parse path params, query params, JSON body, headers
4. **Writing responses** — Status codes, headers, JSON encoding, error helper

**Mini-project: Build a simple "ping/pong" API with a JSON health endpoint and one parameterised route.**

---

### Phase 2 — Middleware & structure (1–2 days)

5. **Middleware pattern** — Write Logger and Recoverer; understand `func(http.Handler) http.Handler`
6. **Handler struct** — Move handlers into a struct with dependencies; wire in main
7. **Project structure** — Set up `cmd/api/`, `internal/handler/`, `internal/store/`

**Mini-project: A "notes" CRUD API — list, create, get, update, delete in-memory notes.**

---

### Phase 3 — Production concerns (2–3 days)

8. **Context** — Pass context to I/O, add timeouts, store values, detect cancellation
9. **Structured logging** — Replace `log.Printf` with `log/slog` JSON logger
10. **Graceful shutdown** — `signal.NotifyContext` + `srv.Shutdown`
11. **Server timeouts** — Add `ReadTimeout`, `WriteTimeout`, `IdleTimeout` to `http.Server`

**Mini-project: Upgrade your notes API with slog middleware, graceful shutdown, and context-aware handlers.**

---

### Phase 4 — Real-world additions (3–5 days)

12. **Testing** — Write handler tests with `httptest.NewRequest` / `httptest.NewRecorder`; aim for >80% coverage
13. **Static files** — Serve an `index.html` + embedded assets
14. **`http.Client`** — Proxy a request to a third-party API (e.g. fetch a GitHub user)
15. **Auth middleware** — Implement JWT validation in middleware; pass user to context
16. **CORS middleware** — Handle preflight OPTIONS correctly

**Capstone project: A "task manager" REST API with:**
- GET/POST/PUT/DELETE `/tasks`
- Bearer token auth middleware
- `log/slog` structured logging
- Graceful shutdown
- Full test coverage with `httptest`
- Static file serving for a minimal HTML frontend
- `http.Client` integration to fetch weather for a task's location

---

### Reference table: Go standard library packages you'll use

| Package | Purpose |
|---|---|
| `net/http` | HTTP server and client |
| `encoding/json` | JSON encoding / decoding |
| `context` | Cancellation, deadlines, values |
| `log/slog` | Structured logging (Go 1.21+) |
| `os/signal` | OS signal handling |
| `syscall` | Signal constants (SIGINT, SIGTERM) |
| `net/http/httptest` | HTTP testing utilities |
| `embed` | Embed files in the binary (Go 1.16+) |
| `strconv` | String ↔ int/float conversion |
| `sync` | Mutex, RWMutex, WaitGroup |
| `time` | Durations, timeouts, timestamps |
| `errors` | `errors.Is`, `errors.As`, `errors.New` |
| `fmt` | Formatted I/O |
| `io` | `io.ReadAll`, `io.Discard`, `io.Copy` |

---

> **Final note:** The Go standard library is genuinely all you need for most APIs. Start without a framework, understand the primitives, and reach for a third-party router only when you hit a real limitation (e.g. complex regex routing, framework-level middleware ecosystems). The patterns in this guide compose cleanly, are easy to test, and will serve you for years.
