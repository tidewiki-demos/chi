# Built-in Middleware

<!-- Maintained by Tidewiki. Edits are kept on later updates; wrap text in tidewiki:keep markers to freeze it. -->

Chi provides a collection of production-ready middleware for common HTTP concerns including logging, compression, authentication, timeouts, and request handling. This page covers the built-in middleware available in the `middleware` package.

## Client IP Detection

Extracting the client's real IP address requires careful handling depending on your network architecture. Chi provides several middleware options to safely extract the client IP from different sources, stored in the request context.

[`middleware/client_ip.go:12-13`](../../middleware/client_ip.go#L12-L13)

**Choose exactly one based on your deployment:**

- [`middleware/client_ip.go:42-55`](../../middleware/client_ip.go#L42-L55) — Use when your reverse proxy sets a dedicated single-IP header (e.g., Nginx with ngx_http_realip_module, Apache with mod_remoteip, or Cloudflare) that **unconditionally overwrites** the value on every request.

- [`middleware/client_ip.go:57-118`](../../middleware/client_ip.go#L57-L118) — Use when you sit behind one or more reverse proxies whose IP ranges you can enumerate as CIDR blocks. The middleware walks the X-Forwarded-For chain right-to-left, skipping trusted entries. Most CDNs publish their IP ranges (Cloudflare, AWS, Fastly, Google Cloud).

- [`middleware/client_ip.go:120-174`](../../middleware/client_ip.go#L120-L174) — Use when you know exactly how many reverse proxies sit between you and the internet, but their IPs are dynamic (autoscaling pools, ephemeral containers). This variant is brittle to architecture changes; prefer explicit CIDRs when possible.

- [`middleware/client_ip.go:176-199`](../../middleware/client_ip.go#L176-L199) — Use when this server is directly connected to the public internet with no reverse proxy in front.

Retrieve the IP in your handler using [`middleware/client_ip.go:201-210`](../../middleware/client_ip.go#L201-L210):

```go
import "github.com/go-chi/chi/v5/middleware"

r.Use(middleware.ClientIPFromHeader("CF-Connecting-IP"))

r.Get("/", func(w http.ResponseWriter, r *http.Request) {
    clientIP := middleware.GetClientIP(r.Context())
    // clientIP is a string, or "" if not set
})
```

These middlewares never mutate `r.RemoteAddr`. The legacy `RealIP` middleware is deprecated due to IP spoofing vulnerabilities.

## Logging and Recovery

### Logger

[`middleware/logger.go:39-41`](../../middleware/logger.go#L39-L41) logs the start and end of each request with status, response size, and elapsed time. When stdout is a TTY, it prints in color; otherwise in black and white.

```go
r := chi.NewRouter()
r.Use(middleware.Logger)      // Must come before Recoverer
r.Use(middleware.Recoverer)
r.Get("/", handler)
```

The logger uses a [`middleware/logger.go:44-59`](../../middleware/logger.go#L44-L59) that accepts a custom `LogFormatter`. The default formatter stores itself in the request context via [`middleware/logger.go:80-84`](../../middleware/logger.go#L80-L84), allowing handlers and other middleware to access and update log entries.

### Recoverer

[`middleware/recoverer.go:22-49`](../../middleware/recoverer.go#L22-L49) recovers from panics, logs them with a formatted backtrace, and returns HTTP 500. It respects `http.ErrAbortHandler` and will not recover from it.

```go
r.Use(middleware.Recoverer)
```

## Authentication

### BasicAuth

[`middleware/basic_auth.go:10-28`](../../middleware/basic_auth.go#L10-L28) implements HTTP Basic Authentication. It compares credentials using constant-time comparison to prevent timing attacks.

```go
creds := map[string]string{
    "user1": "password1",
    "user2": "password2",
}
r.Use(middleware.BasicAuth("MyRealm", creds))
```

## Compression and Content Handling

### Compress

[`middleware/compress.go:45-48`](../../middleware/compress.go#L45-L48) compresses response bodies based on the `Accept-Encoding` request header. It supports gzip and deflate by default, and includes text/markdown, text/csv, text/vtt, application/xml, and text/xml in the default compressible types list.

```go
// Compress level 5 (sensible default), specific content types
r.Use(middleware.Compress(5, "text/html", "text/css", "application/json"))

r.Get("/", func(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    w.Write(jsonData)
})
```

The middleware [`middleware/compress.go:72-142`](../../middleware/compress.go#L72-L142) creates a compressor. Catch-all wildcards ("*/*", "/*") are rejected; pass explicit content types instead. You can [`middleware/compress.go:149-166`](../../middleware/compress.go#L149-L166) to add custom encoders like Brotli.

### AllowContentType

[`middleware/content_type.go:20-44`](../../middleware/content_type.go#L20-L44) enforces a whitelist of request Content-Types, returning 415 Unsupported Media Type if the type doesn't match.

```go
r.Post("/api/data", 
    middleware.AllowContentType("application/json", "application/xml"),
    handler)
```

### AllowContentEncoding

[`middleware/content_encoding.go:10-33`](../../middleware/content_encoding.go#L10-L33) validates that request Content-Encoding headers match a whitelist, returning 415 if they don't.

```go
r.Post("/", middleware.AllowContentEncoding("gzip", "deflate"), handler)
```

### ContentCharset

[`middleware/content_charset.go:9-31`](../../middleware/content_charset.go#L9-L31) validates the charset in Content-Type headers. Requests without a body (ContentLength == 0) are always allowed.

```go
r.Post("/", middleware.ContentCharset("UTF-8", ""), handler)
```

## Request Handling

### Timeout

[`middleware/timeout.go:32-47`](../../middleware/timeout.go#L32-L47) cancels the request context after a given duration and returns HTTP 504 Gateway Timeout if the handler doesn't complete in time. Handlers must check `ctx.Done()` to respect the timeout.

```go
r.Use(middleware.Timeout(30 * time.Second))

r.Get("/long", func(w http.ResponseWriter, r *http.Request) {
    select {
    case <-r.Context().Done():
        return
    case <-time.After(processTime):
        w.Write([]byte("done"))
    }
})
```

### RequestSize

[`middleware/request_size.go:9-17`](../../middleware/request_size.go#L9-L17) limits the maximum request body size using `http.MaxBytesReader`.

```go
r.Use(middleware.RequestSize(1024 * 1024)) // 1MB limit
```

### Throttle

[`middleware/throttle.go:32-34`](../../middleware/throttle.go#L32-L34) limits the number of in-flight requests being processed. It is **not** a per-user rate limiter, but a global ceiling on concurrent requests.

```go
// Allow max 100 concurrent requests
r.Use(middleware.Throttle(100))

// Or with a backlog for pending requests
r.Use(middleware.ThrottleBacklog(100, 50, 60*time.Second))
```

See [`middleware/throttle.go:44-131`](../../middleware/throttle.go#L44-L131) for fine-grained control via `ThrottleOpts`, including custom retry-after headers and status codes.

## Path and URL Handling

### StripSlashes

[`middleware/strip.go:14-34`](../../middleware/strip.go#L14-L34) matches and strips trailing slashes from request paths before routing.

```go
r.Use(middleware.StripSlashes)
r.Get("/users", handler)
// Now matches /users and /users/
```

### RedirectSlashes

[`middleware/strip.go:41-69`](../../middleware/strip.go#L41-L69) redirects requests with trailing slashes to the same path without the slash (HTTP 301).

```go
r.Use(middleware.RedirectSlashes)
```

### StripPrefix

[`middleware/strip.go:73-76`](../../middleware/strip.go#L73-L76) removes a prefix from the request path before routing.

```go
r.Use(middleware.StripPrefix("/api"))
```

### CleanPath

[`middleware/clean_path.go:12-28`](../../middleware/clean_path.go#L12-L28) normalizes double slashes in request paths (e.g., `/users//1` becomes `/users/1`).

```go
r.Use(middleware.CleanPath)
```

### URLFormat

[`middleware/url_format.go:46-77`](../../middleware/url_format.go#L46-L77) parses and strips URL file extensions, storing them in the context for content negotiation.

```go
r.Use(middleware.URLFormat)

r.Get("/articles/{id}", func(w http.ResponseWriter, r *http.Request) {
    format, _ := r.Context().Value(middleware.URLFormatCtxKey).(string)
    switch format {
    case "json":
        render.JSON(w, r, articles)
    case "xml":
        render.XML(w, r, articles)
    }
})
// Now /articles/1, /articles/1.json, /articles/1.xml all match
```

## Request Metadata

### RequestID

[`middleware/request_id.go:67-79`](../../middleware/request_id.go#L67-L79) injects a unique request ID into the request context. It uses the `X-Request-Id` header if present, otherwise generates one.

```go
r.Use(middleware.RequestID)

r.Get("/", func(w http.ResponseWriter, r *http.Request) {
    reqID := middleware.GetReqID(r.Context())
    // Use for logging, tracing, etc.
})
```

## Headers and Routing

### SetHeader

[`middleware/content_type.go:9-16`](../../middleware/content_type.go#L9-L16) sets a response header on all requests.

```go
r.Use(middleware.SetHeader("X-Custom-Header", "value"))
```

### NoCache

[`middleware/nocache.go:40-59`](../../middleware/nocache.go#L40-L59) sets Cache-Control and related headers to prevent response caching by proxies and clients.

```go
r.Use(middleware.NoCache)
```

### Sunset

[`middleware/sunset.go:11-25`](../../middleware/sunset.go#L11-L25) sets `Sunset` and `Deprecation` headers to signal endpoint lifecycle according to RFC 8594.

```go
sunsetAt := time.Date(2025, 12, 31, 0, 0, 0, 0, time.UTC)
r.Use(middleware.Sunset(sunsetAt, "https://example.com/migration"))
```

### RouteHeaders

[`middleware/route_headers.go:42-108`](../../middleware/route_headers.go#L42-L108) routes requests to different middleware based on header values, supporting exact matches and wildcards.

```go
r.Use(middleware.RouteHeaders().
    Route("Host", "api.example.com", apiMiddleware).
    Route("Host", "*.example.com", subdomainMiddleware).
    RouteDefault(defaultMiddleware).
    Handler)
```

## HTTP Method Handling

### GetHead

[`middleware/get_head.go:10-39`](../../middleware/get_head.go#L10-L39) automatically routes undefined HEAD requests to GET handlers, removing the response body as required by the HTTP spec.

```go
r.Use(middleware.GetHead)
r.Get("/users", handler)
// HEAD /users is now routed to the GET handler
```

## Utility Middleware

### Maybe

[`middleware/maybe.go:8-18`](../../middleware/maybe.go#L8-L18) conditionally applies middleware based on a predicate function.

```go
r.Use(middleware.Maybe(
    middleware.Logger,
    func(r *http.Request) bool {
        return r.URL.Path != "/health" // Skip logging /health
    }))
```

### PageRoute

[`middleware/page_route.go:10-20`](../../middleware/page_route.go#L10-L20) routes a static GET request at the middleware level.

```go
r.Use(middleware.PageRoute("/robots.txt", robotsHandler))
```

### Heartbeat

[`middleware/heartbeat.go:12-26`](../../middleware/heartbeat.go#L12-L26) responds immediately to GET/HEAD requests at a specific endpoint with a 200 status and minimal body, useful for load balancer health checks.

```go
r.Use(middleware.Heartbeat("/ping"))
// GET /ping returns 200 with body "."
```

### SupressNotFound

[`middleware/supress_notfound.go:15-27`](../../middleware/supress_notfound.go#L15-L27) quickly responds with a 404 for routes that won't match any handlers, avoiding unnecessary downstream processing. Useful at the top of the middleware stack to filter out bot traffic early.

```go
r.Use(middleware.SupressNotFound(r))
```

### Profiler

[`middleware/profiler.go:22-48`](../../middleware/profiler.go#L22-L48) mounts pprof endpoints for profiling and expvar metrics.

```go
r.Mount("/debug", middleware.Profiler())
```

### WithValue

[`middleware/value.go:9-17`](../../middleware/value.go#L9-L17) sets a context value for all downstream handlers.

```go
r.Use(middleware.WithValue("userID", 123))
```

### PathRewrite

[`middleware/path_rewrite.go:9-16`](../../middleware/path_rewrite.go#L9-L16) rewrites request URL paths using string replacement.

```go
r.Use(middleware.PathRewrite("/old", "/new"))
```

## Decisions

**ContentCharset skips bodyless requests:** ContentCharset middleware now skips validation for requests where ContentLength == 0, allowing GET requests and other methods without a body to pass through even if no Content-Type header is set. This prevents false rejections of legitimate requests like GET with no payload. ([87c34a24a649](https://github.com/go-chi/chi/commit/87c34a24a649))

**Default compressible content types expanded:** Added text/markdown, text/csv, text/vtt, application/xml, and text/xml to the default list because these compress very effectively and are commonly served by web applications. ([d7b767bcbea5](https://github.com/go-chi/chi/commit/d7b767bcbea5), [60ecea54191a](https://github.com/go-chi/chi/commit/60ecea54191a))

**Catch-all compress wildcards rejected:** Passing catch-all patterns like "*/*" or "/*" to NewCompressor now panics at construction instead of silently compressing nothing (or everything). These patterns waste CPU on already-compressed types like zip, jpeg, and png. Users must pass explicit content types instead. ([38939062c5df](https://github.com/go-chi/chi/commit/38939062c5df))

**Brotli compression example updated:** SetEncoder documentation now shows two Brotli implementations: Google's google/brotli/go/cbrotli (with cgo) and andybalholm/brotli (Cgo-free alternative). This gives users flexibility in choosing their compression library based on deployment constraints. ([735ae2b87f8c](https://github.com/go-chi/chi/commit/735ae2b87f8c))

**Flush calls now honor Discard mode:** The WrapResponseWriter implementation was refactored so that Flush() respects the Discard flag, preventing flushes from reaching the original ResponseWriter when discard mode is enabled. Additionally, response status is now properly recorded when flushing occurs before any body is written. ([b1c9ab47626c](https://github.com/go-chi/chi/commit/b1c9ab47626c), [3d1777a1ef88](https://github.com/go-chi/chi/commit/3d1777a1ef88))

**Type signature modernization:** Replaced `interface{}` with `any` throughout middleware packages to align with Go 1.18+ conventions. Updated benchmark loops to use b.Loop() for Go 1.24 compatibility. ([50ef4e4c311f](https://github.com/go-chi/chi/commit/50ef4e4c311f), [756fcb8d630f](https://github.com/go-chi/chi/commit/756fcb8d630f))
