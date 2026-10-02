# Getting Started

<!-- Maintained by Tidewiki. Edits are kept on later updates; wrap text in tidewiki:keep markers to freeze it. -->

chi is a lightweight, idiomatic HTTP router for building Go services. It is built on Go's standard `net/http` package and the `context` package introduced in Go 1.7, making it fully compatible with the Go ecosystem while providing elegant routing and middleware composition.

## Installation

Install chi v5 using `go get`:

```sh
go get -u github.com/go-chi/chi/v5
```

## Core Concepts

chi's design rests on a few key ideas:

**Router**: A `chi.Router` is a composable HTTP handler that matches incoming requests to handlers based on their method and path. [`README.md:177-178`](../../README.md#L177-L178) It's built on a Patricia Radix trie data structure for efficient route matching. [`README.md:177`](../../README.md#L177)

**Middleware**: chi uses standard `net/http` middleware—functions that wrap handlers and execute code before or after them. [`README.md:260-263`](../../README.md#L260-L263) Any middleware compatible with `net/http` works with chi. [`README.md:32`](../../README.md#L32)

**Context Values**: chi leverages Go's `context` package to pass request-scoped values (like user IDs or request IDs) through middleware and handlers without using global state. [`README.md:8-9`](../../README.md#L8-L9)

**URL Parameters**: Routes support named parameters like `/users/{userID}` and wildcards like `/admin/*`. You retrieve these at runtime using `chi.URLParam()`. [`README.md:252-255`](../../README.md#L252-L255)

## Your First Router

Here's a minimal example to get started:

```go
package main

import (
	"net/http"
	"github.com/go-chi/chi/v5"
	"github.com/go-chi/chi/v5/middleware"
)

func main() {
	r := chi.NewRouter()
	
	// Add a middleware for logging
	r.Use(middleware.Logger)
	
	// Define a simple route
	r.Get("/", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte("welcome"))
	})
	
	http.ListenAndServe(":3000", r)
}
```

[`README.md:48-66`](../../README.md#L48-L66)

This creates a router, attaches the built-in Logger middleware, and registers a GET handler on the root path. The router is then passed to `http.ListenAndServe` as a standard `net/http.Handler`.

## Router Methods

The `Router` interface provides methods for defining routes and middleware: [`README.md:185-234`](../../README.md#L185-L234)

- **HTTP method routing**: `Get()`, `Post()`, `Put()`, `Delete()`, `Patch()`, `Query()`, etc., which accept a pattern and handler function
- **Generic routing**: `Handle()` and `HandleFunc()` for handlers matching all HTTP methods
- **Middleware**: `Use()` to add global middleware, `With()` to add inline middleware for specific routes
- **Composition**: `Route()` to mount sub-routers at a path, `Mount()` to attach another `http.Handler`, and `Group()` to organize routes with fresh middleware
- **Error handling**: `NotFound()` and `MethodNotAllowed()` to customize error responses

## Middleware and Handlers

Middleware in chi are standard `net/http` middleware that follow the pattern `func(http.Handler) http.Handler`. [`README.md:269-285`](../../README.md#L269-L285) For example:

```go
// Middleware that sets a context value
func MyMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		ctx := context.WithValue(r.Context(), "user", "123")
		next.ServeHTTP(w, r.WithContext(ctx))
	})
}
```

Handlers then access values set by middleware:

```go
func MyHandler(w http.ResponseWriter, r *http.Request) {
	user := r.Context().Value("user").(string)
	w.Write([]byte("hi " + user))
}
```

[`README.md:295-305`](../../README.md#L295-L305)

## URL Parameters

URL parameters are automatically parsed and stored in the request context. Access them with `chi.URLParam()`:

```go
r.Get("/users/{userID}", func(w http.ResponseWriter, r *http.Request) {
	userID := chi.URLParam(r, "userID")
	w.Write([]byte("user: " + userID))
})
```

[`README.md:314-328`](../../README.md#L314-L328)

Routes also support regex patterns: `/articles/{slug:[a-z-]+}` matches only lowercase letters and hyphens. [`README.md:112`](../../README.md#L112)

## Built-in Middleware

chi includes a `middleware` package with common utilities like `Logger`, `Recoverer` (for panic recovery), `RequestID`, and `Timeout`. [`README.md:333-336`](../../README.md#L333-L336) See [Built-in Middleware](built-in-middleware.md) for details on all available middleware.

A typical middleware stack looks like: [`README.md:88-97`](../../README.md#L88-L97)

```go
r := chi.NewRouter()
r.Use(middleware.RequestID)
r.Use(middleware.Logger)
r.Use(middleware.Recoverer)
r.Use(middleware.Timeout(60 * time.Second))
```

## Decisions

The cite markers in this page point to chi/v5 documentation. This was done in commit 167e1e3bd039 to ensure links resolve to the correct version. The old `github.com/go-chi/chi` module path resolves to stale v1 documentation which lacks some newer middleware like `ClientIP`.

## Next Steps

- [Routing Basics](routing-basics.md) covers route definition patterns and parameters in depth
- [Middleware Fundamentals](middleware-fundamentals.md) explains how to write custom middleware
- [Context Values and Request Handling](context-values.md) details working with context throughout your handlers
- [Built-in Middleware](built-in-middleware.md) lists all included utilities
- [REST API Examples](examples-rest-api.md) shows real-world usage patterns
