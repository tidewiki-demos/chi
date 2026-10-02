# Advanced Middleware Patterns

<!-- Maintained by Tidewiki. Edits are kept on later updates; wrap text in tidewiki:keep markers to freeze it. -->

Build complex request processing workflows by combining chi's middleware primitives. This page covers real IP detection, client identification, request tracking, and conditional middleware execution—essential patterns for production services.

## Real IP Detection

When your service sits behind reverse proxies, accurately identifying the client IP requires careful header parsing. Chi provides multiple strategies depending on your infrastructure.

[`middleware/client_ip.go:19-54`](../../middleware/client_ip.go#L19-L54)

### Single-IP Headers

Use `ClientIPFromHeader` when your reverse proxy sets a dedicated header and **unconditionally overwrites** any client-supplied value on every request. This is the safest approach:

```go
r.Use(middleware.ClientIPFromHeader("CF-Connecting-IP"))

r.Get("/", func(w http.ResponseWriter, r *http.Request) {
    clientIP := middleware.GetClientIP(r.Context())
    // Use clientIP for logging, rate-limiting, etc.
})
```

Examples of safe headers: `X-Real-IP` (Nginx), `X-Client-IP` (Apache), `CF-Connecting-IP` (Cloudflare). **Do not use** `True-Client-IP`, `X-Azure-ClientIP`, or `Fastly-Client-IP` unless your edge strips inbound values—these pass through from the client by default in those products.

[`middleware/client_ip.go:42-54`](../../middleware/client_ip.go#L42-L54)

### X-Forwarded-For with Trusted CIDR Ranges

Use `ClientIPFromXFF` when you can enumerate your proxy IP ranges as CIDR blocks. The middleware walks the X-Forwarded-For chain right-to-left, skipping IPs in trusted ranges, and returns the first untrusted IP:

```go
r.Use(middleware.ClientIPFromXFF(
    "13.32.0.0/15",   // CloudFront IPv4
    "2600:9000::/28", // CloudFront IPv6
))

r.Get("/", func(w http.ResponseWriter, r *http.Request) {
    clientIP := middleware.GetClientIP(r.Context())
})
```

This approach is robust against spoofed IPs prepended by attackers—the outermost proxy (closest to the client) is the only hop that has NOT been under attacker control, so its contribution to the chain is trustworthy. Most CDNs publish their IP ranges: Cloudflare (https://www.cloudflare.com/ips/), AWS (https://ip-ranges.amazonaws.com/ip-ranges.json), Fastly (https://api.fastly.com/public-ip-list), and Google Cloud (https://www.gstatic.com/ipranges/cloud.json).

[`middleware/client_ip.go:57-118`](../../middleware/client_ip.go#L57-L118)

### X-Forwarded-For with Proxy Count

Use `ClientIPFromXFFTrustedProxies` when you know the exact number of trusted proxies but their IPs are dynamic (autoscaling pools, ephemeral containers):

```go
// Exactly 2 proxies between this server and the public internet
r.Use(middleware.ClientIPFromXFFTrustedProxies(2))

r.Get("/", func(w http.ResponseWriter, r *http.Request) {
    clientIP := middleware.GetClientIP(r.Context())
})
```

**Prefer `ClientIPFromXFF` with explicit CIDRs** whenever you can — it cannot off-by-one and is robust to architecture changes. Use this counting variant only when proxy IPs are dynamic and unpublishable.

To verify your count is correct, **send a request from a known IP** and confirm `GetClientIP` returns that IP. If it returns a proxy IP, your count is too LOW — a client can spoof their IP, fix immediately. If it returns "", your count is too HIGH — no leak, but no client IP either.

This middleware reads ONLY X-Forwarded-For; it does not inspect `r.RemoteAddr`. Guarantee at the network layer (security group / firewall) that only your proxies can reach this server.

If the XFF chain has fewer than `numTrustedProxies` entries, no client IP is set (fail-closed).

[`middleware/client_ip.go:120-174`](../../middleware/client_ip.go#L120-L174)

### Direct Internet (No Proxy)

Use `ClientIPFromRemoteAddr` only when this server is directly connected to the public internet with no reverse proxy in front:

```go
r.Use(middleware.ClientIPFromRemoteAddr)

r.Get("/", func(w http.ResponseWriter, r *http.Request) {
    clientIP := middleware.GetClientIP(r.Context())
})
```

Behind a reverse proxy, `RemoteAddr` is the proxy's IP, not the client's.

[`middleware/client_ip.go:176-199`](../../middleware/client_ip.go#L176-L199)

### Retrieving the Client IP

Read the client IP from the context inside handlers using `GetClientIP` (returns string) or `GetClientIPAddr` (returns a typed `netip.Addr`):

```go
r.Get("/admin", func(w http.ResponseWriter, r *http.Request) {
    clientIP := middleware.GetClientIP(r.Context())
    if clientIP == "" {
        http.Error(w, "Client IP not set", http.StatusForbidden)
        return
    }
    
    // Or use typed version for prefix checks:
    addr := middleware.GetClientIPAddr(r.Context())
    if addr.Is6() {
        // Handle IPv6
    }
})
```

[`middleware/client_ip.go:201-219`](../../middleware/client_ip.go#L201-L219)

### Security Properties

**None of the new middlewares mutate `r.RemoteAddr`**—they store the IP in context only. This prevents downstream code from accidentally using a spoofed value if the IP detection logic is misconfigured.

The middlewares use **fail-closed semantics**: if the XFF chain contains unparseable entries, they stop walking rather than falling back to untrusted values. If a header is misconfigured, no IP is set rather than guessing.

IPv4-mapped IPv6 addresses (e.g., `::ffff:1.2.3.4`) are normalized to plain IPv4 before storage and prefix checking, preventing attackers from using notational variants to bypass trust lists. IPv6 zone identifiers are stripped before storage.

## Request ID Tracking

Attach a unique identifier to each request for end-to-end tracing through logs and services:

[`middleware/request_id.go:62-79`](../../middleware/request_id.go#L62-L79)

```go
r.Use(middleware.RequestID)

r.Get("/", func(w http.ResponseWriter, r *http.Request) {
    requestID := middleware.GetReqID(r.Context())
    log.Printf("[%s] Processing request", requestID)
})
```

The middleware checks for an incoming `X-Request-Id` header; if present, it uses that value (allowing trace IDs to flow through your stack). Otherwise, it generates a unique ID combining the hostname, a random process-scoped prefix, and an atomic counter. The generated format is `hostname/random-000001`.

You can customize the header name by setting `middleware.RequestIDHeader` before applying the middleware:

```go
middleware.RequestIDHeader = "X-Trace-Id"
r.Use(middleware.RequestID)
```

Retrieve the ID in handlers with `GetReqID(ctx)`, or generate the next ID in the sequence with `NextRequestID()`.

[`middleware/request_id.go:81-96`](../../middleware/request_id.go#L81-L96)

## Conditional Middleware Execution

Execute a middleware only when a condition is met, allowing sophisticated request routing logic:

[`middleware/maybe.go:5-18`](../../middleware/maybe.go#L5-L18)

```go
r.Use(middleware.Maybe(
    middleware.BasicAuth("admin", "secret123"),
    func(r *http.Request) bool {
        // Only require auth for /admin paths
        return strings.HasPrefix(r.URL.Path, "/admin")
    },
))

r.Get("/admin/users", func(w http.ResponseWriter, r *http.Request) {
    // BasicAuth is required for this route
})

r.Get("/public", func(w http.ResponseWriter, r *http.Request) {
    // BasicAuth is skipped for this route
})
```

The `Maybe` middleware takes two arguments:
1. A middleware function to conditionally apply
2. A predicate function that examines the request and returns true to apply the middleware, false to skip it

This is useful for:
- Rate limiting only external endpoints, not internal admin APIs
- Requiring authentication only on certain path prefixes
- Applying logging or metrics collection only to specific routes

## Decisions

The legacy `RealIP` middleware is deprecated. It was vulnerable to IP spoofing attacks (GHSA-3fxj-6jh8-hvhx, GHSA-rjr7-jggh-pgcp, GHSA-9g5q-2w5x-hmxf) because it:
- Mutated `r.RemoteAddr`, allowing misconfigured header detection to impact all downstream code
- Unconditionally trusted multiple headers in priority order, without requiring the user to choose
- Used the leftmost X-Forwarded-For value instead of the rightmost-untrusted entry

The new `ClientIPFrom*` middlewares fix these issues by:
- Never mutating `r.RemoteAddr`; storing the IP in context only
- Requiring explicit opt-in to a specific source (header name, CIDR list, or proxy count)
- Using rightmost-untrusted logic for XFF chains, which is secure against prepended spoofs

See the example in `middleware/client_ip_example_test.go` for migration guidance. Commit bc02284e9db2 replaced prose documentation on counting proxies with a recipe approach and added a worked example (`Example_clientIPFromXFFTrustedProxies`) showing how client-prepended entries are ignored when using the counting variant with explicit XFF chain structures.
