# Content Encoding and Compression

<!-- Maintained by Tidewiki. Edits are kept on later updates; wrap text in tidewiki:keep markers to freeze it. -->

Handle response compression, request charset validation, content encoding filters, and URL format detection to support flexible API responses based on client capabilities and content types.

## Response Compression

The `Compress` middleware [`middleware/compress.go:34-48`](../../middleware/compress.go#L34-L48) automatically compresses HTTP response bodies based on the client's `Accept-Encoding` header. It supports gzip and deflate encodings by default and can be extended with custom algorithms.

Create a compressor and add it to your router:

```go
package main

import (
	"net/http"
	"github.com/go-chi/chi/v5"
	"github.com/go-chi/chi/v5/middleware"
)

func main() {
	r := chi.NewRouter()
	
	// Compress responses with compression level 5 for these content types
	r.Use(middleware.Compress(5, "text/html", "text/css", "application/json"))
	
	r.Get("/api/data", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		w.Write([]byte(`{"message":"hello"}`))
	})
	
	http.ListenAndServe(":3000", r)
}
```

The middleware [`middleware/compress.go:207-228`](../../middleware/compress.go#L207-L228) wraps response bodies with a compression writer when the client accepts the encoding. It automatically:

- Parses the `Accept-Encoding` request header to determine what algorithms the client supports
- Selects the best encoder based on configured precedence
- Sets the `Content-Encoding` response header and removes `Content-Length` since compressed size is unknown
- Only compresses responses with matching content types

### Compression Levels

Pass a compression level (1-9, where 1 is fastest and 9 is best compression) as the first argument. Level 5 is a sensible default balancing speed and compression ratio. [`middleware/compress.go:44`](../../middleware/compress.go#L44)

### Content Type Filtering

By default, the middleware compresses [`middleware/compress.go:16-32`](../../middleware/compress.go#L16-L32) text/html, text/css, text/plain, text/javascript, text/markdown, text/csv, text/vtt, application/javascript, application/x-javascript, application/json, application/atom+xml, application/rss+xml, application/xml, text/xml, and image/svg+xml.

Specify custom types when creating the middleware:

```go
r.Use(middleware.Compress(5, "application/json", "text/*"))
```

Wildcard patterns like `text/*` are supported [`middleware/compress.go:82-86`](../../middleware/compress.go#L82-L86) to match any subtype. Only the `<type>/*` suffix pattern is allowed. Catch-all wildcards (`*/*` and `/*`) are rejected [`middleware/compress.go:79-85`](../../middleware/compress.go#L79-L85).

### Custom Encoders

Extend the middleware with additional compression algorithms using `SetEncoder`:

```go
compressor := middleware.NewCompressor(5, "application/json")
compressor.SetEncoder("br", func(w io.Writer, level int) io.Writer {
	return cbrotli.NewWriter(w, cbrotli.WriterOptions{Quality: level})
})
r.Use(compressor.Handler)
```

The encoder function receives the response writer and compression level, and returns a writer that compresses data. [`middleware/compress.go:167-203`](../../middleware/compress.go#L167-L203)

### Important: Set Content-Type Header

The middleware only compresses responses with an explicit `Content-Type` header set in the handler [`middleware/compress.go:39-42`](../../middleware/compress.go#L39-L42). If you don't set it, the response body will not be compressed:

```go
r.Get("/data", func(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")  // Required
	w.Write([]byte(`{"status":"ok"}`))
})
```

## Request Charset Validation

The `ContentCharset` middleware [`middleware/content_charset.go:9-32`](../../middleware/content_charset.go#L9-L32) validates that incoming requests specify an acceptable character encoding in their `Content-Type` header, returning 415 Unsupported Media Type if validation fails.

```go
package main

import (
	"net/http"
	"github.com/go-chi/chi/v5"
	"github.com/go-chi/chi/v5/middleware"
)

func main() {
	r := chi.NewRouter()
	
	// Accept only UTF-8 encoded requests
	r.Use(middleware.ContentCharset("UTF-8"))
	
	r.Post("/api/data", func(w http.ResponseWriter, r *http.Request) {
		// Request must have Content-Type: application/json; charset=UTF-8
		w.WriteHeader(http.StatusOK)
	})
	
	http.ListenAndServe(":3000", r)
}
```

The middleware parses the `charset` parameter from the `Content-Type` header and compares it case-insensitively [`middleware/content_charset.go:34-40`](../../middleware/content_charset.go#L34-L40) against allowed values. Requests without a body (`ContentLength == 0`) are always allowed [`middleware/content_charset.go:19-22`](../../middleware/content_charset.go#L19-L22). Pass an empty string to allow requests with no charset specified:

```go
r.Use(middleware.ContentCharset("UTF-8", ""))  // Accept UTF-8 or no charset
```

## Request Content Encoding Filter

The `AllowContentEncoding` middleware [`middleware/content_encoding.go:10-34`](../../middleware/content_encoding.go#L10-L34) enforces a whitelist of request `Content-Encoding` values, rejecting non-empty request bodies with disallowed encodings with a 415 status.

```go
package main

import (
	"net/http"
	"github.com/go-chi/chi/v5"
	"github.com/go-chi/chi/v5/middleware"
)

func main() {
	r := chi.NewRouter()
	
	// Accept only gzip and deflate encoded request bodies
	r.Use(middleware.AllowContentEncoding("gzip", "deflate"))
	
	r.Post("/api/upload", func(w http.ResponseWriter, r *http.Request) {
		// If request has Content-Encoding header, it must be gzip or deflate
		w.WriteHeader(http.StatusOK)
	})
	
	http.ListenAndServe(":3000", r)
}
```

The middleware skips validation for requests with no body (`Content-Length == 0`) [`middleware/content_encoding.go:18-22`](../../middleware/content_encoding.go#L18-L22). All encodings in a request with multiple `Content-Encoding` headers must be in the allowed list.

## URL Format Detection

The `URLFormat` middleware [`middleware/url_format.go:46-77`](../../middleware/url_format.go#L46-L77) extracts file extension suffixes from request paths and stores the format on the request context, allowing handlers to respond in different formats (JSON, XML, etc.) without explicit URL parameters.

```go
package main

import (
	"net/http"
	"github.com/go-chi/chi/v5"
	"github.com/go-chi/chi/v5/middleware"
	"github.com/go-chi/render"
)

func main() {
	r := chi.NewRouter()
	r.Use(middleware.URLFormat)
	
	r.Get("/articles/{id}", func(w http.ResponseWriter, r *http.Request) {
		id := chi.URLParam(r, "id")
		
		// Extract format from context (set by URLFormat middleware)
		format, _ := r.Context().Value(middleware.URLFormatCtxKey).(string)
		
		article := map[string]string{"id": id, "title": "Example"}
		
		switch format {
		case "json":
			render.JSON(w, r, article)
		case "xml":
			render.XML(w, r, article)
		default:
			render.JSON(w, r, article)
		}
	})
	
	http.ListenAndServe(":3000", r)
}
```

Request paths like `/articles/1.json`, `/articles/1.xml`, and `/articles/1` all route to the same handler. The middleware [`middleware/url_format.go:58-70`](../../middleware/url_format.go#L58-L70) parses the extension after the last dot in the path segment following the last slash, stores it in the context under `URLFormatCtxKey`, and rewrites the routing path to remove the suffix.

The middleware operates on the `RoutePath` from chi's route context [`middleware/url_format.go:53-56`](../../middleware/url_format.go#L53-L56), allowing it to work correctly in nested routers and after other middleware has modified the request path.

## Decisions

**Gzip preferred over deflate**: The Compress middleware [`middleware/compress.go:114-129`](../../middleware/compress.go#L114-L129) prioritizes gzip encoding over deflate because older browsers incorrectly handle deflate compression (expecting raw DEFLATE without zlib wrapper). Modern browsers handle both, but gzip is more reliable and consistently implemented across clients.

**Reject catch-all wildcard patterns**: NewCompressor rejects `*/*` and `/*` patterns [`middleware/compress.go:79-85`](../../middleware/compress.go#L79-L85) because compressing every response wastes CPU on already-compressed types like zip, jpeg, and png. Users should pass explicit content types instead.

**Expanded default compressible types**: The default list now includes text/markdown, text/csv, text/vtt, application/xml, and text/xml [`middleware/compress.go:16-32`](../../middleware/compress.go#L16-L32) because these formats compress effectively and are commonly served by web applications.

**Updated Brotli documentation with current packages**: The SetEncoder documentation [`middleware/compress.go:149-166`](../../middleware/compress.go#L149-L166) now references Google's official brotli package (`github.com/google/brotli/go/cbrotli`) and provides an alternative Cgo-free implementation, replacing outdated package references from #1007.

**ContentCharset skips validation for bodyless requests**: The middleware now checks `ContentLength == 0` and skips validation for such requests, fixing the issue where GET requests without a body were incorrectly rejected when they lacked a Content-Type header (#1158).
