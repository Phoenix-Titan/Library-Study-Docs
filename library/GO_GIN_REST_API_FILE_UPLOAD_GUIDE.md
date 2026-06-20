# Go Gin Framework — RESTful API & File Upload Complete Guide

> **Version note:** This guide targets **`github.com/gin-gonic/gin` v1.10+** (stable as of 2026). Go 1.21+ is assumed throughout. Where behavior changed between versions, it's flagged with **⚡ Version note**. Gin's core API has been stable for years — the routing, binding, and middleware patterns here apply from v1.7 onward. Always confirm edge-case behavior against the official repo README or `pkg.go.dev/github.com/gin-gonic/gin` when online.

---

## Table of Contents

1. [What Gin Is & Why Use It](#1-what-gin-is--why-use-it)
2. [Setup & First Server](#2-setup--first-server)
3. [Routing Deep Dive](#3-routing-deep-dive)
4. [The gin.Context](#4-the-gincontext)
5. [Request Binding & Validation](#5-request-binding--validation)
6. [Middleware](#6-middleware)
7. [Full CRUD REST API — Worked Example](#7-full-crud-rest-api--worked-example)
8. [File Uploads (Major Focus)](#8-file-uploads-major-focus)
9. [Serving Static Files & HTML Templates](#9-serving-static-files--html-templates)
10. [Auth with JWT Middleware](#10-auth-with-jwt-middleware)
11. [CORS Setup](#11-cors-setup)
12. [Testing Gin Handlers](#12-testing-gin-handlers)
13. [Idiomatic Project Structure](#13-idiomatic-project-structure)
14. [Tips, Tricks & Gotchas](#14-tips-tricks--gotchas)
15. [Study Path](#15-study-path)

---

## 1. What Gin Is & Why Use It

### What Gin is

**Gin** is a high-performance HTTP web framework written in Go. It wraps Go's standard `net/http` package with a fast router (based on `httprouter`, a radix-tree router), a rich middleware chain, declarative request binding/validation, and a clean `Context` API that eliminates the repetitive boilerplate of raw `net/http`.

```
Request → Router (radix tree) → Middleware chain → Handler → gin.Context helpers → Response
```

### Why Gin over raw net/http

| Feature | `net/http` | Gin |
|---|---|---|
| Router | Pattern-based, no params | Radix tree, `:id`, `*path`, groups |
| Middleware | Manual chaining with wrappers | `Use()`, `Next()`, `Abort()` chain |
| Request binding | Manual `json.Decode` + manual errors | `ShouldBindJSON`, `ShouldBindQuery` with struct tags |
| Validation | None built-in | `binding:"required,email,min=3"` via `go-playground/validator` |
| JSON/XML responses | `json.Marshal` + `w.Write` | `c.JSON(200, obj)` one-liner |
| File uploads | Manual multipart parsing | `c.FormFile`, `c.SaveUploadedFile` |
| Route groups | None | `r.Group("/api/v1")` |
| Performance | Very fast (baseline) | ~40x faster than some frameworks; near-zero allocs on hot paths |

**Raw `net/http` benchmark (no routing, direct handler):**
```
BenchmarkMux  1_000_000   1200 ns/op   416 B/op
BenchmarkGin  1_000_000     75 ns/op     0 B/op  ← near-zero allocations
```

### When to choose Gin

- You need a production REST/JSON API with sane defaults and no ceremony.
- You want middleware (auth, CORS, logging, rate-limiting) in a composable chain.
- You need file upload handling beyond a few lines of boilerplate.
- The team knows Go and wants to stay close to `net/http` concepts but with a better DX.
- You're NOT building something that needs gRPC, GraphQL, or full web MVC — pick the right tool.

> **Alternatives to know:** `Echo` (very similar to Gin, slightly different API), `Chi` (stdlib-compatible, lightweight), `Fiber` (Express-like, but uses fasthttp — not stdlib-compatible). For most new Go APIs in 2026, Gin or Echo are the standard picks.

---

## 2. Setup & First Server

### Prerequisites

```bash
# Go 1.21+
go version  # go version go1.22.x ...

# Initialize a module (do this once per project)
mkdir my-api && cd my-api
go mod init github.com/yourname/my-api
```

### Installing Gin

```bash
go get github.com/gin-gonic/gin
```

This adds Gin and its dependencies (`go-playground/validator`, `ugorji/go/codec`, etc.) to your `go.mod` and `go.sum`.

### First server — `gin.Default()` vs `gin.New()`

```go
// main.go
package main

import (
    "net/http"

    "github.com/gin-gonic/gin"
)

func main() {
    // ─── Option A: gin.Default() ─────────────────────────────────────────────
    // Comes pre-wired with two middleware:
    //   • gin.Logger()   → logs method, path, status, latency to stdout
    //   • gin.Recovery() → catches panics, returns 500 instead of crashing
    // Best for development and most production APIs.
    r := gin.Default()

    // ─── Option B: gin.New() ─────────────────────────────────────────────────
    // Bare engine, zero middleware. You add only what you need.
    // Use when you want full control (e.g., custom structured logger).
    // r := gin.New()
    // r.Use(gin.Recovery())          // always add Recovery in production!
    // r.Use(myStructuredLogger())    // your own logger

    // A simple route
    r.GET("/ping", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{
            "message": "pong",
        })
    })

    // Start listening on :8080
    // r.Run() reads PORT env or defaults to :8080
    // r.Run(":3000")  // or a specific port
    if err := r.Run(":8080"); err != nil {
        panic(err)
    }
}
```

```bash
go run main.go
# [GIN-debug] GET /ping --> main.main.func1 (3 handlers)
# [GIN-debug] Listening on :8080

curl http://localhost:8080/ping
# {"message":"pong"}
```

### Release mode

```go
// In production, set BEFORE creating the engine
gin.SetMode(gin.ReleaseMode)
// or via env:  GIN_MODE=release go run main.go

r := gin.Default()
```

In release mode, Gin suppresses debug route-registration logs and applies minor optimizations.

---

## 3. Routing Deep Dive

### HTTP method handlers

```go
r := gin.Default()

r.GET("/users", listUsers)
r.POST("/users", createUser)
r.PUT("/users/:id", updateUser)
r.PATCH("/users/:id", patchUser)
r.DELETE("/users/:id", deleteUser)
r.HEAD("/users", headUsers)
r.OPTIONS("/users", optionsUsers)

// Handle ANY method
r.Any("/any", func(c *gin.Context) {
    c.String(200, "method: %s", c.Request.Method)
})
```

### Route parameters (`:id`)

```go
// Named parameter — mandatory, one segment
r.GET("/users/:id", func(c *gin.Context) {
    id := c.Param("id")          // "/users/42" → "42"
    c.JSON(200, gin.H{"id": id})
})

// Multiple params
r.GET("/users/:id/posts/:postID", func(c *gin.Context) {
    uid := c.Param("id")
    pid := c.Param("postID")
    c.JSON(200, gin.H{"user": uid, "post": pid})
})
```

### Wildcard routes (`*path`)

```go
// Wildcard — matches everything AFTER the prefix, including slashes
r.GET("/files/*filepath", func(c *gin.Context) {
    fp := c.Param("filepath")    // "/files/images/cat.jpg" → "/images/cat.jpg"
    c.String(200, "file: %s", fp)
})
```

> **Note:** A named param (`:id`) matches exactly one path segment (no `/`). A wildcard (`*path`) matches one or more segments including slashes.

### Query parameters

```go
// URL: /search?q=gin&limit=10&page=2
r.GET("/search", func(c *gin.Context) {
    q     := c.Query("q")                      // "gin" (empty string if missing)
    limit := c.DefaultQuery("limit", "20")     // "10" or default "20"
    page  := c.DefaultQuery("page",  "1")      // "2" or default "1"

    c.JSON(200, gin.H{"q": q, "limit": limit, "page": page})
})
```

### Route groups & API versioning

```go
// All routes under /api/v1 share a common prefix and middleware
v1 := r.Group("/api/v1")
{
    v1.GET("/products",     listProducts)
    v1.POST("/products",    createProduct)
    v1.GET("/products/:id", getProduct)
}

// v2 can evolve independently
v2 := r.Group("/api/v2")
{
    v2.GET("/products", listProductsV2)
}

// Groups can be nested
admin := r.Group("/admin")
admin.Use(AdminAuthMiddleware())
{
    admin.GET("/dashboard", dashboard)

    users := admin.Group("/users")
    {
        users.GET("/",      listAdminUsers)
        users.DELETE("/:id", deleteAdminUser)
    }
}
```

### Router method table

| Method | Use Case |
|---|---|
| `r.GET` | Fetch resource(s) |
| `r.POST` | Create resource |
| `r.PUT` | Full replace |
| `r.PATCH` | Partial update |
| `r.DELETE` | Remove resource |
| `r.Group` | Prefix + shared middleware |
| `r.Use` | Global middleware |
| `r.NoRoute` | Custom 404 handler |
| `r.NoMethod` | Custom 405 handler |

---

## 4. The gin.Context

`*gin.Context` is the core object passed to every handler. It wraps the request, provides response helpers, carries middleware values, and lets you abort the chain.

### Sending responses

```go
func exampleResponses(c *gin.Context) {
    // JSON response (sets Content-Type: application/json)
    c.JSON(http.StatusOK, gin.H{"status": "ok"})

    // Typed struct response (same as JSON — gin.H is just map[string]any)
    type User struct {
        ID   int    `json:"id"`
        Name string `json:"name"`
    }
    c.JSON(200, User{ID: 1, Name: "Alice"})

    // Plain string
    c.String(200, "Hello %s", "World")

    // XML
    c.XML(200, gin.H{"message": "ok"})

    // Render HTML template
    c.HTML(200, "index.html", gin.H{"title": "Home"})

    // Redirect
    c.Redirect(http.StatusFound, "/new-path")

    // File download
    c.File("/path/to/file.pdf")

    // Indented JSON (debug only — slower)
    c.IndentedJSON(200, gin.H{"data": "nice"})
}
```

### Reading the request

```go
func exampleRequest(c *gin.Context) {
    // Route param  → /users/:id
    id := c.Param("id")

    // Query string → ?q=foo&page=1
    q    := c.Query("q")
    page := c.DefaultQuery("page", "1")

    // Header
    auth   := c.GetHeader("Authorization")
    ct     := c.GetHeader("Content-Type")

    // Client IP
    ip := c.ClientIP()

    // Full URL
    fullURL := c.Request.URL.String()

    // Request body (raw) — read once! After binding it's consumed.
    // body, _ := io.ReadAll(c.Request.Body)

    _ = id; _ = q; _ = page; _ = auth; _ = ct; _ = ip; _ = fullURL
}
```

### Aborting the chain

```go
// AbortWithStatus stops the chain and sets status code
c.AbortWithStatus(http.StatusUnauthorized)

// AbortWithStatusJSON stops the chain AND sends a JSON body
c.AbortWithStatusJSON(http.StatusForbidden, gin.H{
    "error": "you do not have permission",
})

// After Abort*, subsequent middleware in the chain is skipped.
// The current handler finishes unless you also return.
```

### Storing & retrieving values across middleware

```go
// In middleware: set a value
c.Set("userID", 42)
c.Set("user", &User{ID: 42, Name: "Alice"})

// In handler: retrieve it
val, exists := c.Get("userID")
if !exists {
    c.AbortWithStatusJSON(401, gin.H{"error": "no user in context"})
    return
}
userID := val.(int)

// Typed getters (added in later versions)
// c.GetString("key"), c.GetInt("key"), c.GetBool("key")
```

### Status code quick reference

| Constant | Code | Meaning |
|---|---|---|
| `http.StatusOK` | 200 | Success |
| `http.StatusCreated` | 201 | Resource created |
| `http.StatusNoContent` | 204 | Success, no body |
| `http.StatusBadRequest` | 400 | Invalid input |
| `http.StatusUnauthorized` | 401 | Not authenticated |
| `http.StatusForbidden` | 403 | Not authorized |
| `http.StatusNotFound` | 404 | Resource not found |
| `http.StatusConflict` | 409 | Conflict (duplicate) |
| `http.StatusUnprocessableEntity` | 422 | Validation failed |
| `http.StatusInternalServerError` | 500 | Server error |

---

## 5. Request Binding & Validation

Gin uses `go-playground/validator/v10` under the hood. Binding reads the request body (JSON, form, query, XML, YAML, etc.) into a struct and validates it in one call.

### JSON body binding

```go
type CreateUserRequest struct {
    Name     string `json:"name"     binding:"required,min=2,max=100"`
    Email    string `json:"email"    binding:"required,email"`
    Age      int    `json:"age"      binding:"required,gte=18,lte=120"`
    Password string `json:"password" binding:"required,min=8"`
    Role     string `json:"role"     binding:"omitempty,oneof=admin user viewer"`
}

func createUser(c *gin.Context) {
    var req CreateUserRequest

    // ShouldBindJSON: does NOT abort on error — you handle the error.
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{
            "error":   "validation failed",
            "details": err.Error(),
        })
        return
    }

    // req.Name, req.Email etc. are now validated and safe to use
    c.JSON(http.StatusCreated, gin.H{"message": "user created", "name": req.Name})
}
```

> **`ShouldBindJSON` vs `BindJSON`:** `BindJSON` calls `c.AbortWithError` automatically on failure and writes a 400 status — you lose control of the error format. Always prefer `ShouldBind*` variants so you can return a consistent error envelope.

### Query parameter binding

```go
type PaginationQuery struct {
    Page  int    `form:"page"   binding:"omitempty,min=1"`
    Limit int    `form:"limit"  binding:"omitempty,min=1,max=100"`
    Sort  string `form:"sort"   binding:"omitempty,oneof=asc desc"`
    Q     string `form:"q"      binding:"omitempty,max=200"`
}

func listProducts(c *gin.Context) {
    var q PaginationQuery
    q.Page  = 1   // set defaults before binding
    q.Limit = 20

    if err := c.ShouldBindQuery(&q); err != nil {
        c.JSON(400, gin.H{"error": err.Error()})
        return
    }
    // q.Page, q.Limit, q.Sort, q.Q are populated
    c.JSON(200, gin.H{"page": q.Page, "limit": q.Limit})
}
```

### Common `binding` tag validators

| Tag | Meaning |
|---|---|
| `required` | Field must be present and non-zero |
| `omitempty` | Skip other validations if value is zero/empty |
| `min=N` | Minimum string length or numeric value |
| `max=N` | Maximum string length or numeric value |
| `gte=N` / `lte=N` | Greater/less-than-or-equal (numeric) |
| `email` | Valid email format |
| `url` | Valid URL |
| `oneof=a b c` | Must be one of the listed values |
| `len=N` | Exact length |
| `numeric` | Contains only digits |
| `alpha` | Contains only letters |
| `alphanum` | Letters and digits only |
| `uuid` | Valid UUID |
| `dive` | Validates each element of a slice/map |

### Custom validators

```go
import (
    "github.com/gin-gonic/gin/binding"
    "github.com/go-playground/validator/v10"
)

// Register once at startup (e.g., in main or init())
func registerCustomValidators() {
    if v, ok := binding.Validator.Engine().(*validator.Validate); ok {
        // Custom: "nospaces" — no whitespace allowed
        v.RegisterValidation("nospaces", func(fl validator.FieldLevel) bool {
            value := fl.Field().String()
            for _, ch := range value {
                if ch == ' ' || ch == '\t' || ch == '\n' {
                    return false
                }
            }
            return true
        })

        // Custom: "slug" — lowercase letters, digits, hyphens only
        v.RegisterValidation("slug", func(fl validator.FieldLevel) bool {
            s := fl.Field().String()
            for _, c := range s {
                if !((c >= 'a' && c <= 'z') || (c >= '0' && c <= '9') || c == '-') {
                    return false
                }
            }
            return len(s) > 0
        })
    }
}

// Now use in structs:
type ProductRequest struct {
    Name string `json:"name" binding:"required,nospaces"`
    Slug string `json:"slug" binding:"required,slug,max=80"`
}
```

### Returning structured validation errors

```go
import "github.com/go-playground/validator/v10"

type FieldError struct {
    Field   string `json:"field"`
    Message string `json:"message"`
}

func formatValidationErrors(err error) []FieldError {
    var ve validator.ValidationErrors
    if !errors.As(err, &ve) {
        return []FieldError{{Field: "body", Message: err.Error()}}
    }

    out := make([]FieldError, 0, len(ve))
    for _, fe := range ve {
        out = append(out, FieldError{
            Field:   fe.Field(),
            Message: validationMessage(fe),
        })
    }
    return out
}

func validationMessage(fe validator.FieldError) string {
    switch fe.Tag() {
    case "required":
        return "this field is required"
    case "email":
        return "must be a valid email address"
    case "min":
        return fmt.Sprintf("minimum length/value is %s", fe.Param())
    case "max":
        return fmt.Sprintf("maximum length/value is %s", fe.Param())
    case "oneof":
        return fmt.Sprintf("must be one of: %s", fe.Param())
    default:
        return fmt.Sprintf("failed validation: %s", fe.Tag())
    }
}

// Usage in handler:
if err := c.ShouldBindJSON(&req); err != nil {
    c.JSON(422, gin.H{
        "error":  "validation_failed",
        "errors": formatValidationErrors(err),
    })
    return
}
```

---

## 6. Middleware

Middleware in Gin is a `gin.HandlerFunc` — the same type as a route handler. The difference is that middleware calls `c.Next()` to pass control down the chain (or `c.Abort()` to halt it).

```
Request
  → Logger middleware (before c.Next())
    → Auth middleware (checks token, may Abort)
      → Your handler
    ← Auth middleware (after c.Next())  ← runs if handler returned normally
  ← Logger middleware (after c.Next()) ← logs duration
Response
```

### Built-in middleware

```go
r := gin.New()                    // bare engine

r.Use(gin.Logger())               // request logging
r.Use(gin.Recovery())             // panic recovery → 500

// Logger options
r.Use(gin.LoggerWithConfig(gin.LoggerConfig{
    SkipPaths: []string{"/health", "/metrics"},
}))
```

### Writing custom middleware

```go
// ─── Request-ID middleware ────────────────────────────────────────────────────
import "github.com/google/uuid"  // go get github.com/google/uuid

func RequestIDMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        requestID := c.GetHeader("X-Request-ID")
        if requestID == "" {
            requestID = uuid.New().String()
        }

        // Store for downstream handlers
        c.Set("requestID", requestID)

        // Echo back in response
        c.Header("X-Request-ID", requestID)

        c.Next() // hand off to next handler
    }
}

// ─── Auth middleware (JWT stub) ───────────────────────────────────────────────
func AuthRequired() gin.HandlerFunc {
    return func(c *gin.Context) {
        token := c.GetHeader("Authorization")

        // Strip "Bearer " prefix
        if len(token) > 7 && token[:7] == "Bearer " {
            token = token[7:]
        }

        if token == "" {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "error": "authorization token required",
            })
            return // stop — don't call c.Next()
        }

        userID, err := validateToken(token) // your JWT validation
        if err != nil {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "error": "invalid or expired token",
            })
            return
        }

        c.Set("userID", userID)
        c.Next() // token valid — continue
    }
}

// ─── CORS middleware (manual version — use gin-contrib/cors in practice) ──────
func CORSMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Header("Access-Control-Allow-Origin",  "*")
        c.Header("Access-Control-Allow-Methods", "GET,POST,PUT,PATCH,DELETE,OPTIONS")
        c.Header("Access-Control-Allow-Headers", "Content-Type,Authorization,X-Request-ID")

        if c.Request.Method == "OPTIONS" {
            c.AbortWithStatus(http.StatusNoContent)
            return
        }
        c.Next()
    }
}
```

### Applying middleware

```go
r := gin.New()
r.Use(gin.Recovery())
r.Use(gin.Logger())
r.Use(RequestIDMiddleware())

// Public routes — no auth
r.GET("/health", healthCheck)
r.POST("/auth/login", login)
r.POST("/auth/register", register)

// Protected group — AuthRequired applied only here
api := r.Group("/api/v1")
api.Use(AuthRequired())
{
    api.GET("/me",           getProfile)
    api.PUT("/me",           updateProfile)
    api.GET("/products",     listProducts)
    api.POST("/products",    createProduct)
    api.DELETE("/products/:id", deleteProduct)
}
```

### Middleware execution order and c.Next()

```go
func TimingMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()

        c.Next() // ← everything BELOW this line runs AFTER the handler

        duration := time.Since(start)
        log.Printf("%s %s took %v", c.Request.Method, c.Request.URL.Path, duration)
    }
}
```

You can write code both before AND after `c.Next()`. Code before runs on the way "in"; code after runs on the way "out" (response already written).

---

## 7. Full CRUD REST API — Worked Example

A complete "products" REST API with in-memory storage, proper DTOs, status codes, and error envelopes. This is the skeleton you'll replace with a real DB.

```go
// products/handler.go
package products

import (
    "errors"
    "net/http"
    "strconv"
    "sync"
    "time"

    "github.com/gin-gonic/gin"
)

// ─── Domain model ─────────────────────────────────────────────────────────────

type Product struct {
    ID          int       `json:"id"`
    Name        string    `json:"name"`
    Description string    `json:"description"`
    Price       float64   `json:"price"`
    Stock       int       `json:"stock"`
    ImageURL    string    `json:"image_url,omitempty"`
    CreatedAt   time.Time `json:"created_at"`
    UpdatedAt   time.Time `json:"updated_at"`
}

// ─── Request / Response DTOs ──────────────────────────────────────────────────

type CreateProductRequest struct {
    Name        string  `json:"name"        binding:"required,min=2,max=200"`
    Description string  `json:"description" binding:"omitempty,max=2000"`
    Price       float64 `json:"price"       binding:"required,gt=0"`
    Stock       int     `json:"stock"       binding:"gte=0"`
}

type UpdateProductRequest struct {
    Name        *string  `json:"name"        binding:"omitempty,min=2,max=200"`
    Description *string  `json:"description" binding:"omitempty,max=2000"`
    Price       *float64 `json:"price"       binding:"omitempty,gt=0"`
    Stock       *int     `json:"stock"       binding:"omitempty,gte=0"`
}

type ListProductsQuery struct {
    Page  int    `form:"page"  binding:"omitempty,min=1"`
    Limit int    `form:"limit" binding:"omitempty,min=1,max=100"`
    Q     string `form:"q"     binding:"omitempty,max=100"`
}

type ErrorResponse struct {
    Error   string `json:"error"`
    Details string `json:"details,omitempty"`
}

// ─── In-memory store ──────────────────────────────────────────────────────────

type Store struct {
    mu       sync.RWMutex
    products map[int]*Product
    nextID   int
}

func NewStore() *Store {
    return &Store{
        products: make(map[int]*Product),
        nextID:   1,
    }
}

var ErrNotFound = errors.New("product not found")

func (s *Store) Create(req CreateProductRequest) *Product {
    s.mu.Lock()
    defer s.mu.Unlock()

    p := &Product{
        ID:          s.nextID,
        Name:        req.Name,
        Description: req.Description,
        Price:       req.Price,
        Stock:       req.Stock,
        CreatedAt:   time.Now(),
        UpdatedAt:   time.Now(),
    }
    s.products[p.ID] = p
    s.nextID++
    return p
}

func (s *Store) GetByID(id int) (*Product, error) {
    s.mu.RLock()
    defer s.mu.RUnlock()

    p, ok := s.products[id]
    if !ok {
        return nil, ErrNotFound
    }
    return p, nil
}

func (s *Store) List(page, limit int, q string) []*Product {
    s.mu.RLock()
    defer s.mu.RUnlock()

    var all []*Product
    for _, p := range s.products {
        if q == "" || containsCI(p.Name, q) {
            all = append(all, p)
        }
    }

    // Simple pagination
    start := (page - 1) * limit
    if start >= len(all) {
        return []*Product{}
    }
    end := start + limit
    if end > len(all) {
        end = len(all)
    }
    return all[start:end]
}

func (s *Store) Update(id int, req UpdateProductRequest) (*Product, error) {
    s.mu.Lock()
    defer s.mu.Unlock()

    p, ok := s.products[id]
    if !ok {
        return nil, ErrNotFound
    }

    if req.Name        != nil { p.Name        = *req.Name }
    if req.Description != nil { p.Description = *req.Description }
    if req.Price       != nil { p.Price       = *req.Price }
    if req.Stock       != nil { p.Stock       = *req.Stock }
    p.UpdatedAt = time.Now()

    return p, nil
}

func (s *Store) Delete(id int) error {
    s.mu.Lock()
    defer s.mu.Unlock()

    if _, ok := s.products[id]; !ok {
        return ErrNotFound
    }
    delete(s.products, id)
    return nil
}

func containsCI(s, sub string) bool {
    return len(s) >= len(sub) // simplified; use strings.Contains + ToLower in real code
}

// ─── Handler ──────────────────────────────────────────────────────────────────

type Handler struct {
    store *Store
}

func NewHandler(store *Store) *Handler {
    return &Handler{store: store}
}

// RegisterRoutes attaches all product routes to a group
func (h *Handler) RegisterRoutes(rg *gin.RouterGroup) {
    rg.GET("",     h.List)
    rg.POST("",    h.Create)
    rg.GET("/:id", h.GetByID)
    rg.PATCH("/:id", h.Update)
    rg.DELETE("/:id", h.Delete)
}

func (h *Handler) List(c *gin.Context) {
    var q ListProductsQuery
    q.Page  = 1
    q.Limit = 20

    if err := c.ShouldBindQuery(&q); err != nil {
        c.JSON(400, ErrorResponse{Error: "invalid query params", Details: err.Error()})
        return
    }

    products := h.store.List(q.Page, q.Limit, q.Q)
    c.JSON(200, gin.H{
        "data":  products,
        "page":  q.Page,
        "limit": q.Limit,
    })
}

func (h *Handler) Create(c *gin.Context) {
    var req CreateProductRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(422, ErrorResponse{Error: "validation failed", Details: err.Error()})
        return
    }

    product := h.store.Create(req)
    c.JSON(http.StatusCreated, product)
}

func (h *Handler) GetByID(c *gin.Context) {
    id, err := strconv.Atoi(c.Param("id"))
    if err != nil {
        c.JSON(400, ErrorResponse{Error: "invalid product id"})
        return
    }

    product, err := h.store.GetByID(id)
    if errors.Is(err, ErrNotFound) {
        c.JSON(404, ErrorResponse{Error: "product not found"})
        return
    }

    c.JSON(200, product)
}

func (h *Handler) Update(c *gin.Context) {
    id, err := strconv.Atoi(c.Param("id"))
    if err != nil {
        c.JSON(400, ErrorResponse{Error: "invalid product id"})
        return
    }

    var req UpdateProductRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(422, ErrorResponse{Error: "validation failed", Details: err.Error()})
        return
    }

    product, err := h.store.Update(id, req)
    if errors.Is(err, ErrNotFound) {
        c.JSON(404, ErrorResponse{Error: "product not found"})
        return
    }

    c.JSON(200, product)
}

func (h *Handler) Delete(c *gin.Context) {
    id, err := strconv.Atoi(c.Param("id"))
    if err != nil {
        c.JSON(400, ErrorResponse{Error: "invalid product id"})
        return
    }

    if err := h.store.Delete(id); errors.Is(err, ErrNotFound) {
        c.JSON(404, ErrorResponse{Error: "product not found"})
        return
    }

    c.JSON(http.StatusNoContent, nil)  // 204 — no body
}
```

```go
// main.go — wiring it together
package main

import (
    "github.com/gin-gonic/gin"
    "github.com/yourname/my-api/products"
)

func main() {
    gin.SetMode(gin.ReleaseMode)

    r := gin.Default()

    store   := products.NewStore()
    handler := products.NewHandler(store)

    v1 := r.Group("/api/v1")
    {
        productsGroup := v1.Group("/products")
        handler.RegisterRoutes(productsGroup)
    }

    r.Run(":8080")
}
```

**Test the API:**
```bash
# Create
curl -X POST http://localhost:8080/api/v1/products \
  -H "Content-Type: application/json" \
  -d '{"name":"Widget Pro","price":29.99,"stock":100}'

# List
curl http://localhost:8080/api/v1/products?page=1&limit=10

# Get one
curl http://localhost:8080/api/v1/products/1

# Update
curl -X PATCH http://localhost:8080/api/v1/products/1 \
  -H "Content-Type: application/json" \
  -d '{"price":24.99}'

# Delete
curl -X DELETE http://localhost:8080/api/v1/products/1
```

---

## 8. File Uploads (Major Focus)

File uploads in Gin use multipart/form-data. The router has a `MaxMultipartMemory` field that controls how much of the upload is buffered in RAM before spilling to a temp file.

### 8.1 Single file upload

```go
func uploadSingleFile(c *gin.Context) {
    // c.FormFile parses multipart form and returns *multipart.FileHeader
    // "file" is the form field name the client sends
    fileHeader, err := c.FormFile("file")
    if err != nil {
        c.JSON(400, gin.H{"error": "file is required: " + err.Error()})
        return
    }

    // c.SaveUploadedFile saves the multipart file to dst path.
    // It creates the file; the directory must already exist.
    dst := "./uploads/" + fileHeader.Filename
    if err := c.SaveUploadedFile(fileHeader, dst); err != nil {
        c.JSON(500, gin.H{"error": "failed to save file"})
        return
    }

    c.JSON(200, gin.H{
        "message":  "file uploaded",
        "filename": fileHeader.Filename,
        "size":     fileHeader.Size,
        "url":      "/uploads/" + fileHeader.Filename,
    })
}
```

**Set max memory BEFORE the upload handler runs (on the router):**
```go
r := gin.Default()
r.MaxMultipartMemory = 8 << 20  // 8 MiB buffered in RAM; rest goes to temp disk
```

### 8.2 Multiple file upload

```go
func uploadMultipleFiles(c *gin.Context) {
    // ParseMultipartForm must have been called — Gin does it lazily when you
    // call c.MultipartForm()
    form, err := c.MultipartForm()
    if err != nil {
        c.JSON(400, gin.H{"error": "invalid multipart form"})
        return
    }

    // form.File is map[string][]*multipart.FileHeader
    // "files" is the form field name
    files := form.File["files"]

    if len(files) == 0 {
        c.JSON(400, gin.H{"error": "at least one file required"})
        return
    }

    uploaded := make([]string, 0, len(files))
    for _, fh := range files {
        dst := "./uploads/" + fh.Filename
        if err := c.SaveUploadedFile(fh, dst); err != nil {
            c.JSON(500, gin.H{"error": "failed to save: " + fh.Filename})
            return
        }
        uploaded = append(uploaded, fh.Filename)
    }

    c.JSON(200, gin.H{"uploaded": uploaded, "count": len(uploaded)})
}
```

### 8.3 Mixing form fields and files

```go
type UploadProductRequest struct {
    Name        string `form:"name"        binding:"required,min=2"`
    Description string `form:"description" binding:"omitempty"`
    Price       float64 `form:"price"      binding:"required,gt=0"`
}

func uploadProductWithImage(c *gin.Context) {
    // 1. Bind non-file fields from the multipart form
    var req UploadProductRequest
    if err := c.ShouldBind(&req); err != nil {
        // ShouldBind reads form fields (text) when Content-Type is multipart
        c.JSON(422, gin.H{"error": err.Error()})
        return
    }

    // 2. Get the image file separately
    fileHeader, err := c.FormFile("image")
    if err != nil {
        c.JSON(400, gin.H{"error": "image file is required"})
        return
    }

    // 3. Validate and save (see Section 8.4 for full validation)
    dst := "./uploads/" + fileHeader.Filename
    if err := c.SaveUploadedFile(fileHeader, dst); err != nil {
        c.JSON(500, gin.H{"error": "failed to save image"})
        return
    }

    c.JSON(201, gin.H{
        "name":      req.Name,
        "price":     req.Price,
        "image_url": "/uploads/" + fileHeader.Filename,
    })
}
```

**curl example — sending file + form fields:**
```bash
curl -X POST http://localhost:8080/api/v1/products/upload \
  -F "name=Widget Pro" \
  -F "price=29.99" \
  -F "image=@/path/to/photo.jpg;type=image/jpeg"
```

### 8.4 Validating uploads (security-critical)

Never trust the client's filename, extension, or Content-Type header. Validate properly:

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
    "strings"
    "time"

    "github.com/google/uuid"
)

// AllowedImageTypes — detected by reading the actual bytes, not the filename.
var AllowedImageMIME = map[string]string{
    "image/jpeg": ".jpg",
    "image/png":  ".png",
    "image/webp": ".webp",
    "image/gif":  ".gif",
}

const MaxImageSize = 5 << 20 // 5 MiB

// ValidateAndSaveImage validates size, MIME type, and saves with a safe unique name.
// Returns the saved path and file URL suffix, or an error.
func ValidateAndSaveImage(fh *multipart.FileHeader, uploadDir string) (string, error) {
    // ── 1. Check file size ───────────────────────────────────────────────────
    if fh.Size > MaxImageSize {
        return "", fmt.Errorf("file too large: max %d bytes, got %d", MaxImageSize, fh.Size)
    }
    if fh.Size == 0 {
        return "", errors.New("file is empty")
    }

    // ── 2. Open the file ─────────────────────────────────────────────────────
    src, err := fh.Open()
    if err != nil {
        return "", fmt.Errorf("cannot open uploaded file: %w", err)
    }
    defer src.Close()

    // ── 3. Detect MIME type by reading the first 512 bytes ───────────────────
    // NEVER trust fh.Header.Get("Content-Type") — it comes from the client.
    buf := make([]byte, 512)
    n, err := src.Read(buf)
    if err != nil && !errors.Is(err, io.EOF) {
        return "", fmt.Errorf("cannot read file: %w", err)
    }
    buf = buf[:n]

    detectedMIME := http.DetectContentType(buf)
    // Strip parameters (e.g. "image/jpeg; charset=utf-8" → "image/jpeg")
    mediaType, _, _ := mime.ParseMediaType(detectedMIME)

    ext, allowed := AllowedImageMIME[mediaType]
    if !allowed {
        return "", fmt.Errorf("file type not allowed: %s (allowed: jpeg, png, webp, gif)", mediaType)
    }

    // ── 4. Generate a safe, unique filename ───────────────────────────────────
    // Never use the client's original filename directly — it can contain
    // path traversal sequences like "../../etc/passwd".
    safeFilename := fmt.Sprintf("%s-%d%s",
        uuid.New().String(),  // globally unique
        time.Now().UnixNano(), // extra entropy
        ext,                  // safe extension from our allowlist
    )

    // ── 5. Ensure upload directory exists ────────────────────────────────────
    if err := os.MkdirAll(uploadDir, 0755); err != nil {
        return "", fmt.Errorf("cannot create upload dir: %w", err)
    }

    // ── 6. Write to disk ─────────────────────────────────────────────────────
    dstPath := filepath.Join(uploadDir, safeFilename)

    // Seek back to the start so we write the full file (we already read 512 bytes)
    if seeker, ok := src.(io.Seeker); ok {
        if _, err := seeker.Seek(0, io.SeekStart); err != nil {
            return "", fmt.Errorf("cannot seek file: %w", err)
        }
    }

    dst, err := os.Create(dstPath)
    if err != nil {
        return "", fmt.Errorf("cannot create dest file: %w", err)
    }
    defer dst.Close()

    if _, err := io.Copy(dst, src); err != nil {
        os.Remove(dstPath) // clean up partial write
        return "", fmt.Errorf("cannot write file: %w", err)
    }

    return safeFilename, nil
}
```

**Using it in a handler:**
```go
func (h *Handler) UploadImage(c *gin.Context) {
    fh, err := c.FormFile("image")
    if err != nil {
        c.JSON(400, gin.H{"error": "image field required"})
        return
    }

    filename, err := upload.ValidateAndSaveImage(fh, "./uploads/images")
    if err != nil {
        // Return 422 for validation errors, 500 for I/O errors
        // In production, inspect error type more carefully
        c.JSON(422, gin.H{"error": err.Error()})
        return
    }

    imageURL := "/uploads/images/" + filename
    c.JSON(201, gin.H{"url": imageURL})
}
```

### 8.5 Validating it's a real image (stdlib)

`http.DetectContentType` above uses magic bytes. For a stricter check (ensuring the image is fully decodable, not just a JPEG header over random data), decode it with the `image` package:

```go
import (
    "image"
    _ "image/gif"   // register decoder
    _ "image/jpeg"  // register decoder
    _ "image/png"   // register decoder
    _ "golang.org/x/image/webp" // go get golang.org/x/image
)

// ValidateImageDecodable tries to fully decode the image.
// Call this on the opened multipart file (seek to 0 first).
func ValidateImageDecodable(r io.Reader) (width, height int, err error) {
    cfg, format, err := image.DecodeConfig(r)
    if err != nil {
        return 0, 0, fmt.Errorf("not a valid image (%w)", err)
    }
    _ = format // "jpeg", "png", etc.

    // Sanity check: reject suspiciously large canvas (image bomb protection)
    if cfg.Width > 10000 || cfg.Height > 10000 {
        return 0, 0, errors.New("image dimensions too large")
    }

    return cfg.Width, cfg.Height, nil
}
```

### 8.6 Optional image resize / thumbnail

```go
// go get github.com/disintegration/imaging
import "github.com/disintegration/imaging"

func createThumbnail(srcPath, dstPath string, width int) error {
    // Open the saved image
    src, err := imaging.Open(srcPath)
    if err != nil {
        return err
    }

    // Resize to width pixels (height auto-calculated, preserving aspect ratio)
    thumb := imaging.Resize(src, width, 0, imaging.Lanczos)

    // Save as JPEG (quality 85)
    return imaging.Save(thumb, dstPath, imaging.JPEGQuality(85))
}

// In handler, after saving:
thumbPath := "./uploads/thumbs/thumb_" + filename
if err := createThumbnail("./uploads/images/"+filename, thumbPath, 300); err != nil {
    log.Printf("warning: thumbnail creation failed: %v", err)
    // Non-fatal — don't fail the request over a thumbnail
}
```

**Alternative (stdlib only — no dependencies):**
```go
import (
    "image"
    "image/jpeg"
    _ "image/png"
    "golang.org/x/image/draw"
)

func resizeStdlib(src image.Image, newWidth int) image.Image {
    bounds := src.Bounds()
    origW, origH := bounds.Max.X, bounds.Max.Y
    newH := (origH * newWidth) / origW

    dst := image.NewRGBA(image.Rect(0, 0, newWidth, newH))
    draw.BiLinear.Scale(dst, dst.Bounds(), src, bounds, draw.Over, nil)
    return dst
}
```

### 8.7 Storing to S3 (brief overview)

For production, store uploads in object storage rather than local disk:

```go
// go get github.com/aws/aws-sdk-go-v2/...
import (
    "context"
    "github.com/aws/aws-sdk-go-v2/aws"
    "github.com/aws/aws-sdk-go-v2/config"
    "github.com/aws/aws-sdk-go-v2/service/s3"
)

func uploadToS3(ctx context.Context, client *s3.Client, bucket, key string, body io.Reader, contentType string) (string, error) {
    _, err := client.PutObject(ctx, &s3.PutObjectInput{
        Bucket:      aws.String(bucket),
        Key:         aws.String(key),
        Body:        body,
        ContentType: aws.String(contentType),
    })
    if err != nil {
        return "", err
    }
    return fmt.Sprintf("https://%s.s3.amazonaws.com/%s", bucket, key), nil
}

// Same pattern works for Cloudflare R2, MinIO, Backblaze B2, DigitalOcean Spaces
// (all expose an S3-compatible API)
```

### 8.8 Serving uploaded files & returning the URL

```go
r := gin.Default()

// Serve the entire ./uploads directory at the /uploads URL path.
// Access: GET /uploads/images/abc123.jpg
r.Static("/uploads", "./uploads")

// Or use StaticFS for an embedded filesystem (Go 1.16+ embed):
// import "embed"
// //go:embed uploads
// var uploadedFiles embed.FS
// r.StaticFS("/uploads", http.FS(uploadedFiles))
```

```go
// Build the URL to return to clients:
func buildUploadURL(c *gin.Context, filename string) string {
    scheme := "http"
    if c.Request.TLS != nil {
        scheme = "https"
    }
    return fmt.Sprintf("%s://%s/uploads/images/%s", scheme, c.Request.Host, filename)
}
```

### 8.9 Complete upload handler wired up

```go
// main.go excerpt
r := gin.Default()
r.MaxMultipartMemory = 8 << 20  // 8 MiB

// Serve uploads
r.Static("/uploads", "./uploads")

// Upload endpoints
api := r.Group("/api/v1")
api.POST("/upload/image",  uploadImageHandler)
api.POST("/upload/images", uploadMultipleImagesHandler)

func uploadImageHandler(c *gin.Context) {
    fh, err := c.FormFile("image")
    if err != nil {
        c.JSON(400, gin.H{"error": "image field required"})
        return
    }

    filename, err := upload.ValidateAndSaveImage(fh, "./uploads/images")
    if err != nil {
        c.JSON(422, gin.H{"error": err.Error()})
        return
    }

    c.JSON(201, gin.H{
        "url":      buildUploadURL(c, filename),
        "filename": filename,
        "size":     fh.Size,
    })
}

func uploadMultipleImagesHandler(c *gin.Context) {
    form, err := c.MultipartForm()
    if err != nil {
        c.JSON(400, gin.H{"error": "invalid multipart form"})
        return
    }

    files := form.File["images"]
    if len(files) == 0 {
        c.JSON(400, gin.H{"error": "at least one image required"})
        return
    }
    if len(files) > 10 {
        c.JSON(400, gin.H{"error": "maximum 10 images per request"})
        return
    }

    urls := make([]string, 0, len(files))
    for _, fh := range files {
        filename, err := upload.ValidateAndSaveImage(fh, "./uploads/images")
        if err != nil {
            c.JSON(422, gin.H{"error": fmt.Sprintf("file %s: %v", fh.Filename, err)})
            return
        }
        urls = append(urls, buildUploadURL(c, filename))
    }

    c.JSON(201, gin.H{"urls": urls, "count": len(urls)})
}
```

---

## 9. Serving Static Files & HTML Templates

### Static files

```go
// Serve a directory
r.Static("/assets", "./static")       // /assets/style.css → ./static/style.css
r.StaticFile("/favicon.ico", "./static/favicon.ico")

// Embedded FS (Go 1.16+)
import "embed"
//go:embed static
var staticFiles embed.FS
r.StaticFS("/static", http.FS(staticFiles))
```

### HTML templates

```go
// Load all templates matching the glob
r.LoadHTMLGlob("templates/**/*")
// OR load specific files:
// r.LoadHTMLFiles("templates/index.html", "templates/product.html")

r.GET("/", func(c *gin.Context) {
    c.HTML(http.StatusOK, "index.html", gin.H{
        "title":    "My Store",
        "products": []string{"Widget", "Gadget"},
    })
})
```

```html
<!-- templates/index.html -->
<!DOCTYPE html>
<html>
<head><title>{{ .title }}</title></head>
<body>
  <h1>{{ .title }}</h1>
  <ul>
    {{ range .products }}
      <li>{{ . }}</li>
    {{ end }}
  </ul>
</body>
</html>
```

> **Tip:** For production apps, use a frontend framework (React, Vue, Svelte) and serve Gin purely as an API. Reserve HTML templates for simple admin panels or emails.

---

## 10. Auth with JWT Middleware

```bash
go get github.com/golang-jwt/jwt/v5
```

```go
package auth

import (
    "errors"
    "net/http"
    "strings"
    "time"

    "github.com/gin-gonic/gin"
    "github.com/golang-jwt/jwt/v5"
)

var jwtSecret = []byte("your-256-bit-secret-from-env") // load from os.Getenv in production

type Claims struct {
    UserID int    `json:"user_id"`
    Email  string `json:"email"`
    Role   string `json:"role"`
    jwt.RegisteredClaims
}

// GenerateToken creates a signed JWT for a user
func GenerateToken(userID int, email, role string) (string, error) {
    claims := Claims{
        UserID: userID,
        Email:  email,
        Role:   role,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(24 * time.Hour)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
            Issuer:    "my-api",
        },
    }

    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(jwtSecret)
}

// JWTMiddleware validates Bearer token and sets claims in context
func JWTMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        authHeader := c.GetHeader("Authorization")
        if authHeader == "" || !strings.HasPrefix(authHeader, "Bearer ") {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "error": "authorization header required (Bearer <token>)",
            })
            return
        }

        tokenStr := strings.TrimPrefix(authHeader, "Bearer ")

        claims := &Claims{}
        token, err := jwt.ParseWithClaims(tokenStr, claims, func(t *jwt.Token) (any, error) {
            if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
                return nil, errors.New("unexpected signing method")
            }
            return jwtSecret, nil
        })

        if err != nil || !token.Valid {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "error": "invalid or expired token",
            })
            return
        }

        // Store claims for downstream handlers
        c.Set("userID", claims.UserID)
        c.Set("email",  claims.Email)
        c.Set("role",   claims.Role)
        c.Next()
    }
}

// AdminOnly middleware — chain after JWTMiddleware
func AdminOnly() gin.HandlerFunc {
    return func(c *gin.Context) {
        role, _ := c.Get("role")
        if role != "admin" {
            c.AbortWithStatusJSON(http.StatusForbidden, gin.H{
                "error": "admin access required",
            })
            return
        }
        c.Next()
    }
}
```

```go
// Wiring it up
func main() {
    r := gin.Default()

    // Public
    r.POST("/auth/login",    loginHandler)
    r.POST("/auth/register", registerHandler)

    // Protected
    protected := r.Group("/api/v1")
    protected.Use(auth.JWTMiddleware())
    {
        protected.GET("/me", meHandler)
        protected.GET("/products", listProducts)

        // Admin only — nested group
        admin := protected.Group("/admin")
        admin.Use(auth.AdminOnly())
        {
            admin.DELETE("/users/:id", deleteUser)
        }
    }

    r.Run(":8080")
}

func loginHandler(c *gin.Context) {
    // ... validate credentials ...
    token, err := auth.GenerateToken(1, "alice@example.com", "user")
    if err != nil {
        c.JSON(500, gin.H{"error": "token generation failed"})
        return
    }
    c.JSON(200, gin.H{"token": token})
}

func meHandler(c *gin.Context) {
    userID, _ := c.Get("userID")
    email,  _ := c.Get("email")
    c.JSON(200, gin.H{"user_id": userID, "email": email})
}
```

---

## 11. CORS Setup

```bash
go get github.com/gin-contrib/cors
```

```go
import "github.com/gin-contrib/cors"

r := gin.Default()

// ── Simple: allow all origins (development only!) ─────────────────────────────
r.Use(cors.Default())  // allows all origins, GET/POST/PUT/PATCH/DELETE/HEAD

// ── Production: explicit configuration ───────────────────────────────────────
r.Use(cors.New(cors.Config{
    AllowOrigins:     []string{"https://myapp.com", "https://admin.myapp.com"},
    AllowMethods:     []string{"GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"},
    AllowHeaders:     []string{"Origin", "Content-Type", "Authorization", "X-Request-ID"},
    ExposeHeaders:    []string{"Content-Length", "X-Request-ID"},
    AllowCredentials: true,          // allow cookies / Authorization header
    MaxAge:           12 * time.Hour, // preflight cache duration
}))

// ── Dynamic origin validation ─────────────────────────────────────────────────
r.Use(cors.New(cors.Config{
    AllowOriginFunc: func(origin string) bool {
        // Allow any subdomain of myapp.com
        return strings.HasSuffix(origin, ".myapp.com") || origin == "https://myapp.com"
    },
    AllowMethods: []string{"GET", "POST", "PUT", "DELETE", "OPTIONS"},
    AllowHeaders: []string{"Content-Type", "Authorization"},
}))
```

---

## 12. Testing Gin Handlers

Gin ships in test mode, which suppresses log output during tests. Use Go's `net/http/httptest` for handler-level tests — no real server needed.

```go
// products/handler_test.go
package products_test

import (
    "bytes"
    "encoding/json"
    "net/http"
    "net/http/httptest"
    "testing"

    "github.com/gin-gonic/gin"
    "github.com/yourname/my-api/products"
)

func setupTestRouter() *gin.Engine {
    gin.SetMode(gin.TestMode) // suppress logs in tests

    r := gin.New()  // gin.New() not gin.Default() (no Logger noise in test output)
    r.Use(gin.Recovery())

    store   := products.NewStore()
    handler := products.NewHandler(store)

    v1 := r.Group("/api/v1")
    handler.RegisterRoutes(v1.Group("/products"))

    return r
}

func TestCreateProduct(t *testing.T) {
    r := setupTestRouter()

    // Build the request body
    body := map[string]any{
        "name":  "Test Widget",
        "price": 19.99,
        "stock": 50,
    }
    bodyBytes, _ := json.Marshal(body)

    // Create a test request + response recorder
    req  := httptest.NewRequest(http.MethodPost, "/api/v1/products", bytes.NewBuffer(bodyBytes))
    req.Header.Set("Content-Type", "application/json")
    w := httptest.NewRecorder()

    // Serve the request
    r.ServeHTTP(w, req)

    // Assert status
    if w.Code != http.StatusCreated {
        t.Errorf("expected 201, got %d; body: %s", w.Code, w.Body.String())
    }

    // Parse response
    var resp map[string]any
    if err := json.Unmarshal(w.Body.Bytes(), &resp); err != nil {
        t.Fatal("response is not valid JSON")
    }

    if resp["name"] != "Test Widget" {
        t.Errorf("expected name 'Test Widget', got %v", resp["name"])
    }
}

func TestCreateProduct_ValidationError(t *testing.T) {
    r := setupTestRouter()

    // Missing required "name" field
    body := map[string]any{"price": -1} // also invalid price
    bodyBytes, _ := json.Marshal(body)

    req := httptest.NewRequest(http.MethodPost, "/api/v1/products", bytes.NewBuffer(bodyBytes))
    req.Header.Set("Content-Type", "application/json")
    w := httptest.NewRecorder()

    r.ServeHTTP(w, req)

    if w.Code != http.StatusUnprocessableEntity {
        t.Errorf("expected 422, got %d", w.Code)
    }
}

func TestGetProduct_NotFound(t *testing.T) {
    r := setupTestRouter()

    req := httptest.NewRequest(http.MethodGet, "/api/v1/products/999", nil)
    w   := httptest.NewRecorder()

    r.ServeHTTP(w, req)

    if w.Code != http.StatusNotFound {
        t.Errorf("expected 404, got %d", w.Code)
    }
}

// Testing file uploads
func TestUploadImage_NoFile(t *testing.T) {
    r := setupTestRouter()
    // Attach the upload handler if not already registered in setupTestRouter

    req := httptest.NewRequest(http.MethodPost, "/api/v1/upload/image", nil)
    req.Header.Set("Content-Type", "multipart/form-data")
    w := httptest.NewRecorder()

    r.ServeHTTP(w, req)

    if w.Code != http.StatusBadRequest {
        t.Errorf("expected 400 when no file, got %d", w.Code)
    }
}
```

```bash
go test ./... -v
go test ./products/... -run TestCreate -v
go test ./... -count=1   # disable test cache
```

---

## 13. Idiomatic Project Structure

```
my-api/
├── cmd/
│   └── server/
│       └── main.go                  # Entry point: wire up dependencies, start server
│
├── internal/                        # Private application code (cannot be imported externally)
│   ├── config/
│   │   └── config.go                # Load env vars (os.Getenv or godotenv)
│   │
│   ├── middleware/
│   │   ├── auth.go                  # JWT middleware
│   │   ├── cors.go                  # CORS config
│   │   ├── logger.go                # Structured logger
│   │   └── requestid.go             # X-Request-ID
│   │
│   ├── products/
│   │   ├── handler.go               # gin.HandlerFunc methods
│   │   ├── handler_test.go          # httptest-based tests
│   │   ├── model.go                 # Product struct, DTOs
│   │   ├── repository.go            # DB interface + implementations
│   │   └── service.go               # Business logic (optional layer)
│   │
│   ├── users/
│   │   ├── handler.go
│   │   ├── model.go
│   │   └── repository.go
│   │
│   ├── upload/
│   │   ├── upload.go                # ValidateAndSaveImage, thumbnail, etc.
│   │   └── upload_test.go
│   │
│   └── server/
│       └── server.go                # gin.Engine setup, route registration
│
├── uploads/                         # Runtime upload directory (gitignore this)
│   ├── images/
│   └── thumbs/
│
├── static/                          # Static assets served by r.Static
│
├── templates/                       # HTML templates (if used)
│
├── go.mod
├── go.sum
├── .env                             # Local env (never commit secrets)
└── .gitignore
```

**`cmd/server/main.go` — clean entry point:**
```go
package main

import (
    "log"

    "github.com/yourname/my-api/internal/config"
    "github.com/yourname/my-api/internal/server"
)

func main() {
    cfg := config.Load() // loads from env / .env file

    srv := server.New(cfg)

    log.Printf("starting server on %s", cfg.Addr)
    if err := srv.Run(); err != nil {
        log.Fatalf("server error: %v", err)
    }
}
```

**`internal/server/server.go` — engine setup:**
```go
package server

import (
    "github.com/gin-gonic/gin"
    "github.com/gin-contrib/cors"
    "github.com/yourname/my-api/internal/config"
    "github.com/yourname/my-api/internal/middleware"
    "github.com/yourname/my-api/internal/products"
)

type Server struct {
    engine *gin.Engine
    cfg    *config.Config
}

func New(cfg *config.Config) *Server {
    gin.SetMode(cfg.GinMode)

    r := gin.New()
    r.Use(gin.Recovery())
    r.Use(middleware.Logger())
    r.Use(cors.New(cfg.CORSConfig()))
    r.Use(middleware.RequestID())

    r.MaxMultipartMemory = cfg.MaxUploadSize

    // Health check (no auth)
    r.GET("/health", func(c *gin.Context) { c.JSON(200, gin.H{"status": "ok"}) })

    // API routes
    api := r.Group("/api/v1")

    // Products
    productStore   := products.NewStore()
    productHandler := products.NewHandler(productStore)
    productHandler.RegisterRoutes(api.Group("/products"))

    return &Server{engine: r, cfg: cfg}
}

func (s *Server) Run() error {
    return s.engine.Run(s.cfg.Addr)
}
```

---

## 14. Tips, Tricks & Gotchas

### Upload security

- **Never use the client's filename.** `fh.Filename` can be `../../etc/cron.d/evil`. Always generate your own name (UUID + extension from allowlist).
- **Detect MIME by bytes, not extension.** Read the first 512 bytes and call `http.DetectContentType`. An attacker can rename `malicious.php` to `photo.jpg`.
- **Set `MaxMultipartMemory` before any upload handler.** Default is 32 MiB. Set it lower for image-only APIs.
- **Limit file count per request.** Check `len(form.File["files"])` and reject if over your limit.
- **Scan for image bombs.** After decoding, check `cfg.Width * cfg.Height` — a 1-byte PNG can decode to a 50,000 x 50,000 canvas and exhaust RAM.
- **Store outside the webroot.** Don't put uploads inside a Go-embedded static dir. Serve them explicitly, so a `.go` file upload can't be executed.

### Body size limits

```go
// MaxMultipartMemory controls multipart form RAM buffer.
// For a hard total body limit (including non-upload endpoints), use a middleware:
import "github.com/gin-contrib/size"
r.Use(size.RequestSizeLimiter(10 << 20)) // 10 MiB hard limit for ALL requests
```

### Binding pitfalls

- **`ShouldBind` reads the body ONCE.** If you call it twice, the second call gets nothing (body is a stream). Use `c.ShouldBindBodyWith` (from `gin-contrib/body-limit` or by buffering manually) if you need to read the body in middleware AND handler.
- **`binding:"required"` on int/bool matches zero value as missing.** If `0` is a valid value for a field, use a pointer (`*int`) and `omitempty` instead of `required`.
- **Form vs JSON binding:** `c.ShouldBind` auto-detects based on `Content-Type`. For `application/json` use `ShouldBindJSON` explicitly to avoid surprises.
- **Query binding uses `form` tag**, not `json`. `c.ShouldBindQuery` reads the URL query string only.

### gin.SetMode

```go
// MUST be called before gin.Default() or gin.New()
gin.SetMode(gin.ReleaseMode)   // production: suppresses debug logs
gin.SetMode(gin.TestMode)      // tests: suppresses all output
gin.SetMode(gin.DebugMode)     // default: verbose route listing
```

Or via environment: `GIN_MODE=release ./my-api`

### c.Next() / c.Abort() rules

- Calling `c.Abort()` does NOT stop the current function — you must `return` after it.
- Middleware that calls `c.Next()` executes code BEFORE next handler on the way in, and AFTER on the way out. This is how timing, logging, and cleanup middleware works.
- `c.AbortWithStatusJSON` writes the response headers — do NOT call `c.JSON` after it in the same handler, or you'll get a "superfluous response.WriteHeader call" warning.

### Panic safety

- Always use `gin.Recovery()` (or `gin.Default()` which includes it). Without it, a panic in any handler crashes the whole process.
- In release mode, Recovery returns a 500 with no stack trace in the body — good for security.

### Returning errors consistently

Define a single error envelope and use it everywhere:
```go
type APIError struct {
    Code    string `json:"code"`
    Message string `json:"message"`
}

func respondError(c *gin.Context, status int, code, msg string) {
    c.JSON(status, APIError{Code: code, Message: msg})
}
```

### Hot reload in development

```bash
go install github.com/air-verse/air@latest
air  # watches for file changes and rebuilds
```

### ⚡ Version notes

- **Gin v1.9+** improved `ShouldBindBodyWith` to use a pool — less allocation.
- **Gin v1.10+** added `c.ContextWithFallback` for passing Go context with deadlines through the chain; use `c.Request.Context()` for database timeouts.
- `go-playground/validator` v10 is the current version bundled with Gin; v9 is EOL.

---

## 15. Study Path

Work through these in order. Each project forces you to learn the relevant Gin concepts.

### Phase 1 — Foundations (Week 1-2)

**Read/do:**
1. Official Gin README examples (`github.com/gin-gonic/gin`)
2. Go Tour (if you haven't): `tour.golang.org`
3. Sections 1–4 of this guide

**Project 1 — Hello API**
Build a bare-bones API with 3 routes:
- `GET /ping` → `{"message": "pong"}`
- `GET /users/:id` → hardcoded user JSON
- `POST /echo` → echo the JSON body back

Goals: routing, `gin.Context`, `c.JSON`, `c.Param`.

---

### Phase 2 — CRUD + Binding (Week 2-3)

**Read/do:**
- Sections 5–7 of this guide
- `go-playground/validator` docs

**Project 2 — Products CRUD API**
Implement the full Products API from Section 7:
- In-memory store (sync.RWMutex map)
- Proper DTOs with validation
- 404 / 422 / 500 error envelopes
- List with query-param pagination

Goals: binding, validation, error handling, status codes.

---

### Phase 3 — Auth & Middleware (Week 3-4)

**Read/do:**
- Sections 6, 10, 11
- `golang-jwt/jwt` docs

**Project 3 — Auth-Protected API**
Add to Project 2:
- `POST /auth/register` + `POST /auth/login` (return JWT)
- JWT middleware protecting `/api/v1/*`
- Request-ID middleware
- CORS middleware (allow `localhost:3000`)

Goals: middleware chains, JWT, route groups.

---

### Phase 4 — File Uploads (Week 4-5)

**Read/do:**
- Section 8 of this guide (all subsections, including security)

**Project 4 — Image Upload API** ← THE flagship project

Build an API that:
1. Accepts `POST /api/v1/upload/image` (single image) and `/api/v1/upload/images` (up to 5)
2. Validates: max 5 MiB, only JPEG/PNG/WebP, real image (decode check)
3. Generates UUID filename, saves to `./uploads/images/`
4. Creates a 300px thumbnail in `./uploads/thumbs/` (use `disintegration/imaging`)
5. Serves uploads via `r.Static("/uploads", "./uploads")`
6. Returns `{"url": "...", "thumbnail_url": "...", "width": N, "height": N}`
7. Protected by JWT (only logged-in users can upload)

Test with: `curl -F "image=@photo.jpg" -H "Authorization: Bearer <token>" http://localhost:8080/api/v1/upload/image`

---

### Phase 5 — Production Polish (Week 5-6)

**Read/do:**
- Section 12 (testing), 13 (project structure), 14 (gotchas)
- `pkg.go.dev/github.com/gin-gonic/gin` — skim all exported types

**Project 5 — Production-Ready Refactor**
Refactor Project 4 into the idiomatic structure from Section 13:
1. Split into `internal/` packages
2. Add `cmd/server/main.go`
3. Write handler tests (httptest) for CRUD + upload
4. Add `air` for hot reload
5. Add a `Makefile` with `make run`, `make test`, `make build`
6. Set `GIN_MODE=release` via `.env`
7. Add a `GET /health` endpoint for load balancer checks

---

### Reference bookmarks (offline-friendly)

| Resource | Location |
|---|---|
| Gin docs | `pkg.go.dev/github.com/gin-gonic/gin` |
| Validator tags | `pkg.go.dev/github.com/go-playground/validator/v10` |
| golang-jwt | `pkg.go.dev/github.com/golang-jwt/jwt/v5` |
| gin-contrib/cors | `pkg.go.dev/github.com/gin-contrib/cors` |
| disintegration/imaging | `pkg.go.dev/github.com/disintegration/imaging` |
| Go image stdlib | `pkg.go.dev/image` |
| httptest | `pkg.go.dev/net/http/httptest` |
| Go standard library | `pkg.go.dev/std` |

---

*Guide accurate as of June 2026 — Gin v1.10+, Go 1.22+.*
