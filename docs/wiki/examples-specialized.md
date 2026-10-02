# Specialized Examples

<!-- Maintained by Tidewiki. Edits are kept on later updates; wrap text in tidewiki:keep markers to freeze it. -->

This page covers advanced usage patterns for chi: custom HTTP methods, request limits, routing introspection, and custom handler signatures. These examples build on concepts from [Routing Basics](routing-basics.md) and [Middleware Fundamentals](middleware-fundamentals.md).

## Custom HTTP Methods

By default, chi provides convenience methods like `Get()`, `Post()`, and `Delete()`. To use a custom or less common HTTP method, use the `Method()` function:

```go
r := chi.NewRouter()
r.Method("CUSTOM", "/path", myHandler)
```

[`_examples/custom-handler/main.go:22`](../../_examples/custom-handler/main.go#L22) shows registering a route with `Method()`. This is useful for non-standard methods or when you need to support methods beyond the built-in shortcuts.

## Custom Handler Signatures

Standard Go handlers have the signature `func(http.ResponseWriter, *http.Request)`. If you want handlers that return errors, you can wrap them by implementing `http.Handler`:

[`_examples/custom-handler/main.go:10-18`](../../_examples/custom-handler/main.go#L10-L18)

```go
type Handler func(w http.ResponseWriter, r *http.Request) error

func (h Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	if err := h(w, r); err != nil {
		// Handle error centrally
		w.WriteHeader(503)
		w.Write([]byte("error"))
	}
}

r := chi.NewRouter()
r.Method("GET", "/", Handler(customHandler))
```

This pattern lets handlers return errors, which are then handled uniformly by the wrapper. It centralizes error handling without middleware overhead.

## Request Limits: Timeout and Throttle

### Timeout

[`_examples/limits/main.go:42-44`](../../_examples/limits/main.go#L42-L44) shows using `middleware.Timeout()` to cancel request processing if it exceeds a duration. The server responds with `http.StatusGatewayTimeout` (504):

```go
r.Group(func(r chi.Router) {
	r.Use(middleware.Timeout(2500 * time.Millisecond))
	r.Get("/slow", mySlowHandler)
})
```

Handlers should monitor `r.Context().Done()` to exit early when the timeout fires.

### Throttle

[`_examples/limits/main.go:68-70`](../../_examples/limits/main.go#L68-L70) shows using `middleware.Throttle()` to limit concurrent requests on a route:

```go
r.Group(func(r chi.Router) {
	r.Use(middleware.Throttle(1)) // Only 1 request at a time
	r.Get("/expensive", myExpensiveHandler)
})
```

Requests beyond the limit are queued and processed when a slot becomes available. [`_examples/limits/main.go:71-89`](../../_examples/limits/main.go#L71-L89) shows checking for context cancellation when throttled requests timeout.

See [Built-in Middleware](built-in-middleware.md) for more details on these functions.

## Routing Introspection with chi.Walk

To discover all routes in a router programmatically, use `chi.Walk()`:

[`_examples/router-walk/main.go:28-36`](../../_examples/router-walk/main.go#L28-L36)

```go
walkFunc := func(method string, route string, handler http.Handler, 
                 middlewares ...func(http.Handler) http.Handler) error {
	fmt.Printf("%s %s\n", method, route)
	return nil
}

if err := chi.Walk(r, walkFunc); err != nil {
	log.Fatal(err)
}
```

`chi.Walk()` traverses the router tree and calls your function for each route, passing the HTTP method, route pattern, handler, and middleware chain. This is useful for:

- Generating API documentation
- Building route lists for monitoring dashboards
- Validating router structure in tests
- Printing debug information

The route pattern uses `/*/` as a wildcard placeholder (see [`_examples/router-walk/main.go:29`](../../_examples/router-walk/main.go#L29) for how to normalize it).

## Decisions

No architectural decisions are documented for this page.
