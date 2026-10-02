# Security and Request Filtering

<!-- Maintained by Tidewiki. Edits are kept on later updates; wrap text in tidewiki:keep markers to freeze it. -->

Apply security headers, control request sizes, manage rate limiting, and filter requests based on headers or content types. Chi provides built-in middleware to protect your API and enforce security policies.

## Security Headers

### NoCache

[`middleware/nocache.go:31-39`](../../middleware/nocache.go#L31-L39) prevents responses from being cached by proxies and clients. It sets HTTP headers that signal no caching should occur, and removes any ETag headers that might enable conditional requests.

```go
import "github.com/go-chi/chi/v5/middleware"

r := chi.NewRouter()
r.Use(middleware.NoCache)
r.Get("/api/data", handler)
```

### SetHeader

[`middleware/content_type.go:8-16`](../../middleware/content_type.go#L8-L16) is a utility to set arbitrary response headers on all requests passing through a route or router.

```go
r.Use(middleware.SetHeader("X-Content-Type-Options", "nosniff"))
r.Use(middleware.SetHeader("X-Frame-Options", "DENY"))
```

## Request Size Limiting

### RequestSize

[`middleware/request_size.go:7-18`](../../middleware/request_size.go#L7-L18) limits the maximum size of incoming request bodies. If a client sends a body larger than the specified limit, the request is rejected. This protects against accidental or malicious large uploads.

```go
r := chi.NewRouter()
// Limit requests to 10MB
r.Use(middleware.RequestSize(10 * 1024 * 1024))
r.Post("/upload", uploadHandler)
```

The middleware wraps the request body with `http.MaxBytesReader`, which transparently enforces the limit during reading.

## Content Type Filtering

### AllowContentType

[`middleware/content_type.go:18-44`](../../middleware/content_type.go#L18-L44) enforces a whitelist of allowed `Content-Type` headers on incoming requests. Requests with disallowed content types receive a 415 Unsupported Media Type response. [`middleware/content_type.go:28-31`](../../middleware/content_type.go#L28-L31) Requests with empty bodies bypass the check.

```go
r := chi.NewRouter()
// Only accept JSON and XML on this endpoint
r.Post("/api/data", 
    middleware.AllowContentType("application/json", "application/xml")(handler))
```

The middleware normalizes content types by converting to lowercase and extracting the main type before the semicolon (e.g., `"application/json; charset=UTF-8"` becomes `"application/json"`).

## Rate Limiting and Request Throttling

### Throttle

[`middleware/throttle.go:28-34`](../../middleware/throttle.go#L28-L34) limits the number of requests being processed concurrently. It places a hard ceiling on in-flight requests across all clients, not per-user. When the limit is reached, new requests receive a 429 Too Many Requests response.

```go
r := chi.NewRouter()
// Process maximum 50 concurrent requests
r.Use(middleware.Throttle(50))
r.Get("/api/resource", handler)
```

### ThrottleBacklog

[`middleware/throttle.go:36-41`](../../middleware/throttle.go#L36-L41) extends throttling with a backlog queue. Requests beyond the immediate limit wait in a queue for up to the specified timeout. If they don't get a processing slot within that time, they receive a timeout error.

```go
r := chi.NewRouter()
// 10 concurrent slots, 50 waiting in backlog for max 30 seconds
r.Use(middleware.ThrottleBacklog(10, 50, 30*time.Second))
r.Get("/api/resource", handler)
```

[`middleware/throttle.go:44-131`](../../middleware/throttle.go#L44-L131) `ThrottleWithOpts` provides fine-grained control:

```go
r.Use(middleware.ThrottleWithOpts(middleware.ThrottleOpts{
    Limit:          10,
    BacklogLimit:   50,
    BacklogTimeout: 30 * time.Second,
    StatusCode:     http.StatusServiceUnavailable, // Custom error code
    RetryAfterFn: func(ctxDone bool) time.Duration {
        return 60 * time.Second
    },
}))
```

The `RetryAfterFn` callback sets the `Retry-After` header to tell clients when to retry. It receives a boolean indicating whether the context was canceled (`true`) or the backlog timeout was exceeded (`false`).

## Header-Based Request Routing

### RouteHeaders

[[cite:middleware/route_headers.go:8-26, 42-44]] allows you to apply different middleware based on request header values. This enables scenarios like subdomain-specific handlers or CORS policies that differ by origin.

```go
r := chi.NewRouter()
apiRouter := chi.NewRouter()
appRouter := chi.NewRouter()

r.Use(middleware.RouteHeaders().
    Route("Host", "api.example.com", 
        middleware.New(apiRouter)).
    Route("Host", "app.example.com", 
        middleware.New(appRouter)).
    Handler)
```

[`middleware/route_headers.go:48-69`](../../middleware/route_headers.go#L48-L69) Pattern matching supports exact matches and wildcards:

- `"example.com"` matches exactly
- `"*.example.com"` matches any subdomain
- `"api.*"` matches `api.` followed by anything
- `"*"` matches anything (useful as a default)

```go
r.Use(middleware.RouteHeaders().
    Route("Origin", "https://trusted.example.com", trustedCORS).
    Route("Origin", "*.example.com", internalCORS).
    RouteDefault(publicCORS).
    Handler)
```

[`middleware/route_headers.go:58-70`](../../middleware/route_headers.go#L58-L70) `RouteAny` matches multiple patterns for the same header:

```go
r.Use(middleware.RouteHeaders().
    RouteAny("Content-Type", 
        []string{"application/json", "application/xml"}, 
        jsonXmlHandler).
    Handler)
```

## Basic Authentication

### BasicAuth

[`middleware/basic_auth.go:9-28`](../../middleware/basic_auth.go#L9-L28) implements HTTP Basic Authentication. It validates incoming requests against a map of username-password credentials and sends a `WWW-Authenticate` header on failure, prompting clients to authenticate.

```go
r := chi.NewRouter()
credentials := map[string]string{
    "alice": "secure-password-123",
    "bob":   "another-password",
}
r.Use(middleware.BasicAuth("My API", credentials))
r.Get("/admin", adminHandler)
```

[`middleware/basic_auth.go:20`](../../middleware/basic_auth.go#L20) Password comparison uses `subtle.ConstantTimeCompare` to prevent timing attacks, ensuring that incorrect passwords take the same time to reject as correct ones.

On authentication failure, the server responds with status 401 and sets the `WWW-Authenticate` header to `Basic realm="My API"`.

## Decisions

**Constant-time password comparison:** BasicAuth uses `subtle.ConstantTimeCompare` [`middleware/basic_auth.go:20`](../../middleware/basic_auth.go#L20) rather than simple string equality to prevent timing-based attacks where attackers could infer password length or content from response time differences.
