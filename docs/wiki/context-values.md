# Context Values and Request Handling

<!-- Maintained by Tidewiki. Edits are kept on later updates; wrap text in tidewiki:keep markers to freeze it. -->

Chi uses Go's `context.Context` package to manage request-scoped data. This page explains how to extract URL parameters, access the routing context, and pass values through middleware chains.

## URL Parameters

URL parameters are captured during route matching and stored in chi's routing context. Extract them using the helper functions provided by chi.

### Extracting Parameters

Use `URLParam()` to get a parameter from the current request:

```go
func MyHandler(w http.ResponseWriter, r *http.Request) {
	userID := chi.URLParam(r, "userID")
	// userID contains the matched parameter value
	fmt.Fprintf(w, "User: %s\n", userID)
}

r := chi.NewRouter()
r.Get("/users/{userID}", MyHandler)
```

If you already have access to the context directly (for example, in middleware or spawned goroutines), use `URLParamFromCtx()`:

```go
userID := chi.URLParamFromCtx(ctx, "userID")
```

Both functions [`context.go:10-13`](../../context.go#L10-L13) [`context.go:15-21`](../../context.go#L15-L21) return an empty string if the parameter is not found.

## Routing Context

Chi stores routing information in a custom `Context` object attached to the request's `context.Context` under the key `RouteCtxKey`. This context tracks matched patterns, URL parameters, and other routing metadata.

### Accessing the Routing Context

Retrieve chi's routing context with `RouteContext()`:

```go
func MyMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		rctx := chi.RouteContext(r.Context())
		// rctx is now available for inspection
		next.ServeHTTP(w, r)
	})
}
```

[`context.go:23-28`](../../context.go#L23-L28) This function safely extracts the routing context and returns `nil` if none is set.

### Route Pattern

After a request is routed, you can retrieve the complete matched route pattern. This is useful for logging, metrics, and telemetry.

**Important:** Call `RoutePattern()` after the handler chain completes, typically in a logging middleware that wraps the next handler. The pattern changes as the request passes through nested routers.

```go
func Instrument(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		next.ServeHTTP(w, r)
		
		rctx := chi.RouteContext(r.Context())
		pattern := rctx.RoutePattern()
		// pattern is now the final matched route, e.g., "/users/{userID}"
		log.Printf("Route: %s", pattern)
	})
}
```

[`context.go:147-158`](../../context.go#L147-L158) The `RoutePattern()` method joins all matched patterns across nested routers and cleans up intermediate wildcards.

## Passing Values Through Middleware

Use the standard `context.WithValue()` to attach application-specific data to a request:

```go
func AuthMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// Extract user from request (e.g., from a token)
		userID := "user123"
		
		// Attach to context
		ctx := context.WithValue(r.Context(), "userID", userID)
		
		next.ServeHTTP(w, r.WithContext(ctx))
	})
}

func MyHandler(w http.ResponseWriter, r *http.Request) {
	userID := r.Context().Value("userID").(string)
	fmt.Fprintf(w, "User: %s\n", userID)
}
```

This pattern is standard Go and works seamlessly with chi's middleware chain. See [Middleware Fundamentals](middleware-fundamentals.md) for more on building middleware.

## Context Structure

[`context.go:40-79`](../../context.go#L40-L79) The `Context` struct stores:

- **URLParams**: A stack of all captured parameters across nested routers
- **RoutePatterns**: All matched patterns throughout the request lifecycle
- **RoutePath** and **RouteMethod**: Optionally overridden for internal route searching
- **Routes**: The available routes at the current level

The context is managed by chi and reset after each request. You typically do not interact with these fields directly; use the helper functions instead.

## URL Parameters Structure

[`context.go:170-173`](../../context.go#L170-L173) `RouteParams` is a simple struct holding parallel `Keys` and `Values` slices. The `URLParam()` method searches from the end backward, allowing nested routers to shadow parameters from parent routers.

## Getting the Complete Route Pattern

When using nested routers (via `Mount()` or `Route()`), route patterns accumulate. The `RoutePattern()` method cleans these up by removing intermediate wildcards:

```go
r := chi.NewRouter()
r.Route("/v1", func(r chi.Router) {
	r.Mount("/resources", resourcesController().Router())
})

// Inside resourcesController:
// r.Get("/{resourceID}", handler)
// 
// In handler, RoutePattern() returns: "/v1/resources/{resourceID}"
// Not: "/v1/*resources/*/{resourceID}"
```

This behavior is tested to ensure patterns remain clean across router hierarchies.

## Cloning Routing Context

The `Clone()` method creates a deep copy of a routing context for use outside the request/response lifecycle, such as in goroutines that outlive the request handler:

```go
func MyHandler(w http.ResponseWriter, r *http.Request) {
	rctx := chi.RouteContext(r.Context())
	ctxCopy := rctx.Clone()
	
	go func() {
		// Safe to use ctxCopy after handler returns
		userID := ctxCopy.URLParam("userID")
	}()
	
	w.WriteHeader(http.StatusOK)
}
```

[`context.go:98-120`](../../context.go#L98-L120) `Clone()` must be called before the request handler returns; after that, the original context is reset and reused by another request. The method deep-copies all reference fields (slices, maps, and pointers) and detaches the clone from the request's context, which would be canceled once the request finishes.

## Decisions

**Clone method added** (commit 15767233591f): The `Clone()` method enables safe use of routing contexts in asynchronous operations that outlive the request handler. This is useful for background tasks, mirror testing, and other scenarios where the context needs to persist after the request is served. The method must be called before the handler returns to avoid racing with context reset and reuse.

**Simplify chi.URLParam** (commit ae6be7469132): The `URLParam()` function now delegates to `URLParamFromCtx()` to avoid duplicate implementations. The internal `URLParam` method on the `Context` type was also optimized to use `slices.Backward()` for cleaner backward iteration over URL parameter keys.
