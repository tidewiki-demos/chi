# Routing Basics

<!-- Maintained by Tidewiki. Edits are kept on later updates; wrap text in tidewiki:keep markers to freeze it. -->

Chi provides a simple, composable routing system for defining HTTP endpoints. Routes are defined using URL patterns that support named parameters, wildcards, and specific HTTP methods.

## Creating a Router

Start by creating a new router with [`chi.go:59-62`](../../chi.go#L59-L62):

```go
r := chi.NewRouter()
```

The `Router` interface [`chi.go:66-114`](../../chi.go#L66-L114) defines all the routing methods you'll use. Most commonly, you'll interact with a `*Mux` returned by `NewRouter()`, which implements this interface.

## Defining Routes with HTTP Methods

Chi provides convenience methods for each standard HTTP method. Register a route and its handler like this:

```go
r.Get("/users", getUsersHandler)
r.Post("/users", createUserHandler)
r.Put("/users/{id}", updateUserHandler)
r.Delete("/users/{id}", deleteUserHandler)
```

Available methods are [`chi.go:96-105`](../../chi.go#L96-L105): `Get`, `Post`, `Put`, `Patch`, `Delete`, `Head`, `Options`, `Connect`, and `Trace`.

Alternatively, use `Method()` to specify any supported HTTP method:

```go
r.Method("PATCH", "/users/{id}", patchUserHandler)
```

For a route that accepts all HTTP methods, use `Handle()` or `HandleFunc()`:

```go
r.HandleFunc("/ping", func(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("pong"))
})
```

See [`_examples/hello-world/main.go:16-18`](../../_examples/hello-world/main.go#L16-L18) for a basic example.

## Named Parameters

URL patterns can include named parameters enclosed in braces. [`chi.go:34-36`](../../chi.go#L34-L36) A simple named placeholder `{name}` matches any sequence of characters up to the next `/` or end of the URL.

```go
r.Get("/users/{userID}", func(w http.ResponseWriter, r *http.Request) {
    userID := r.PathValue("userID")
    w.Write([]byte("User ID: " + userID))
})
```

This pattern matches `/users/123` but not `/users/123/profile`.

See [`_examples/pathvalue/main.go:14-24`](../../_examples/pathvalue/main.go#L14-L24) for a complete example.

## Regular Expression Constraints

[`chi.go:38-42`](../../chi.go#L38-L42) You can constrain a parameter to match a regular expression by using the syntax `{name:regexp}`. The forward slash `/` is never matched by the expression.

```go
r.Get("/articles/{year:\\d{4}}/{month:\\d{2}}", func(w http.ResponseWriter, r *http.Request) {
    year := r.PathValue("year")
    month := r.PathValue("month")
    // Handle article filtering by year and month
})
```

This matches `/articles/2024/03` but not `/articles/2024/3`.

An anonymous regexp pattern is also allowed using an empty name:

```go
r.Get("/files/{:\\d+}", handler) // matches /files/42
```

## Wildcards

[`chi.go:44-46`](../../chi.go#L44-L46) The asterisk `*` matches the rest of the requested URL, including forward slashes. Any trailing characters in the pattern are ignored. This is useful for catch-all routes or mounting subrouters.

```go
r.Get("/page/*", func(w http.ResponseWriter, r *http.Request) {
    rest := r.PathValue("*")
    // rest could be "intro/latest" for request "/page/intro/latest"
})
```

## Custom HTTP Methods

If you need to support custom HTTP methods beyond the standard ones, register them first [`_examples/custom-method/main.go:10-14`](../../_examples/custom-method/main.go#L10-L14), then use them with `Method()` or `MethodFunc()`:

```go
chi.RegisterMethod("LINK")
chi.RegisterMethod("UNLINK")

r.MethodFunc("LINK", "/link", func(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("custom link method"))
})
```

See [`_examples/custom-method/main.go`](../../_examples/custom-method/main.go) for the complete example.

## Pattern Matching Examples

[`chi.go:48-54`](../../chi.go#L48-L54) demonstrates how different patterns are matched:

- `/user/{name}` matches `/user/jsmith` but not `/user/jsmith/info`
- `/user/{name}/info` matches `/user/jsmith/info`
- `/page/*` matches `/page/intro/latest`
- `/date/{yyyy:\d\d\d\d}/{mm:\d\d}/{dd:\d\d}` matches `/date/2017/04/01`

## Important Pattern Rules

[`chi.go:30-32`](../../chi.go#L30-L32) All URL patterns must begin with a slash. Trailing slashes must be handled explicitly—`/users` and `/users/` are distinct patterns.

```go
r.Get("/users", handler1)    // handles GET /users
r.Get("/users/", handler2)   // handles GET /users/
```

For more details on how route matching works internally, see [Route Matching and Tree](route-matching-tree.md).
