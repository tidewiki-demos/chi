# Middleware Fundamentals

<!-- Maintained by Tidewiki. Edits are kept on later updates; wrap text in tidewiki:keep markers to freeze it. -->

Middleware in chi is a way to wrap HTTP handlers and intercept or modify requests and responses. Chi uses the standard Go `net/http` middleware pattern—a function that takes an `http.Handler` and returns an `http.Handler`—making it compatible with any middleware designed for the standard library.

## The Middleware Pattern

A middleware is a function with this signature:

```go
func(next http.Handler) http.Handler
```

It receives the next handler in the chain and returns a new handler. When the new handler is called, it can do work before calling `next.ServeHTTP()`, after it returns, or both.

**Example middleware that logs request duration:**

```go
func loggingMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		next.ServeHTTP(w, r)
		log.Printf("%s %s took %v", r.Method, r.URL.Path, time.Since(start))
	})
}
```

## Middleware Chains

Chi provides a `Chain()` function to compose multiple middlewares and attach them to a handler. [`chain.go:5-8`](../../chain.go#L5-L8)

```go
func Chain(middlewares ...func(http.Handler) http.Handler) Middlewares
```

It returns a `Middlewares` type with two methods to build handlers:

- `Handler(h http.Handler) http.Handler` — wraps an `http.Handler` [`chain.go:12-14`](../../chain.go#L12-L14)
- `HandlerFunc(h http.HandlerFunc) http.Handler` — wraps an `http.HandlerFunc` [`chain.go:18-20`](../../chain.go#L18-L20)

**Usage example:**

```go
import "github.com/go-chi/chi/v5"

handler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
})

// Apply middleware chain
chain := chi.Chain(loggingMiddleware, authMiddleware)
finalHandler := chain.HandlerFunc(handler)

// Register with a route
r := chi.NewRouter()
r.Handle("/path", finalHandler)
```

## Execution Order

Middlewares are applied in reverse order of the chain. The rightmost middleware is closest to the endpoint handler.

[`chain.go:36-48`](../../chain.go#L36-L48) shows how the chain builds: it starts with the last middleware wrapping the endpoint, then each preceding middleware wraps the result. This means for `Chain(a, b, c).Handler(endpoint)`, the execution order is:

```
a → b → c → endpoint
```

When a request arrives, `a` runs first, then `b`, then `c`, then `endpoint`. On the way back, `c` completes, then `b`, then `a`.

## Wrapping Existing Handlers

The `middleware.New()` helper converts an `http.Handler` into middleware form:

[`middleware/middleware.go:5-12`](../../middleware/middleware.go#L5-L12)

```go
func New(h http.Handler) func(next http.Handler) http.Handler
```

This is useful when you have a handler you want to insert into a middleware chain.

**Example:**

```go
import "github.com/go-chi/chi/v5/middleware"

existingHandler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("Before endpoint"))
})

// Convert to middleware
asMiddleware := middleware.New(existingHandler)

// Use in a chain
r := chi.NewRouter()
r.Use(asMiddleware)
```

## Applying Middleware to Routes

Chi routers accept middleware via the `Use()` method. Middleware applied to a router affects all handlers registered on that router and its subrouters. You can also wrap individual route handlers with chains.

**Per-router (affects all routes):**

```go
r := chi.NewRouter()
r.Use(loggingMiddleware)
r.Use(authMiddleware)
r.Get("/", handler)
```

**Per-route (affects only that handler):**

```go
r := chi.NewRouter()
r.Get("/", chi.Chain(loggingMiddleware, authMiddleware).HandlerFunc(handler))
```

## Standard Library Compatibility

Because chi uses the standard `net/http` middleware pattern, any third-party middleware written for Go's standard library works directly:

```go
import "github.com/some/third-party/middleware"

r := chi.NewRouter()
r.Use(middleware.SomeMiddleware())
r.Use(middleware.AnotherMiddleware())
```

See [Built-in Middleware](built-in-middleware.md) for chi's provided middleware and [Advanced Middleware Patterns](middleware-advanced.md) for complex composition techniques.
