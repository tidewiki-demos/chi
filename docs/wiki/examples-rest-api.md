# REST API Examples

<!-- Maintained by Tidewiki. Edits are kept on later updates; wrap text in tidewiki:keep markers to freeze it. -->

Production-ready examples demonstrating modular API design, resource routing, versioning, and graceful shutdown patterns in chi.

## Modular REST API with CRUD Operations

The core REST example [`_examples/rest/main.go`](../../_examples/rest/main.go) shows a complete article management service with proper resource routing, middleware composition, and error handling.

**Key patterns:**

- **Modular routing with `Route()`**: [`_examples/rest/main.go:79-93`](../../_examples/rest/main.go#L79-L93) groups all `/articles` endpoints under a subrouter, reducing boilerplate and clarifying intent.

```go
r.Route("/articles", func(r chi.Router) {
    r.With(paginate).Get("/", ListArticles)
    r.Post("/", CreateArticle)
    r.Get("/search", SearchArticles)
    
    r.Route("/{articleID}", func(r chi.Router) {
        r.Use(ArticleCtx)
        r.Get("/", GetArticle)
        r.Put("/", UpdateArticle)
        r.Delete("/", DeleteArticle)
    })
})
```

- **Context-based resource loading**: [`_examples/rest/main.go:124-145`](../../_examples/rest/main.go#L124-L145) The `ArticleCtx` middleware loads an article by ID or slug from the URL and stores it on the request context, so handlers don't repeat database lookups. This pattern keeps handlers focused on business logic.

```go
func ArticleCtx(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        var article *Article
        if articleID := chi.URLParam(r, "articleID"); articleID != "" {
            article, _ = dbGetArticle(articleID)
        }
        ctx := context.WithValue(r.Context(), "article", article)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

- **Request/response separation**: [`_examples/rest/main.go:310-376`](../../_examples/rest/main.go#L310-L376) Separate `ArticleRequest` and `ArticleResponse` types let you control input validation, field transformation, and output enrichment without coupling to the data model. The `Bind()` method runs post-unmarshal, and `Render()` runs before encoding.

- **Error responses**: [`_examples/rest/main.go:396-433`](../../_examples/rest/main.go#L396-L433) The `ErrResponse` type implements `render.Renderer` to provide consistent error payloads with HTTP status codes and application-level error details.

## Sub-Router Architecture

The `_examples/todos-resource` demonstrates how to organize larger services as independent resource routers. [`_examples/todos-resource/main.go:26-27`](../../_examples/todos-resource/main.go#L26-L27) mounts resource handlers:

```go
r.Mount("/users", usersResource{}.Routes())
r.Mount("/todos", todosResource{}.Routes())
```

Each resource is a separate type with a `Routes()` method [`_examples/todos-resource/todos.go:12-29`](../../_examples/todos-resource/todos.go#L12-L29) that returns a configured subrouter. This keeps resource-specific middleware and handlers isolated and testable.

## Admin Sub-Router with Middleware

[`_examples/rest/main.go:219-244`](../../_examples/rest/main.go#L219-L244) shows a complete separate admin router mounted under `/admin` with its own middleware chain. The `AdminOnly` middleware [`_examples/rest/main.go:235-243`](../../_examples/rest/main.go#L235-L243) restricts access by checking a context value:

```go
func adminRouter() chi.Router {
    r := chi.NewRouter()
    r.Use(AdminOnly)
    r.Get("/", func(w http.ResponseWriter, r *http.Request) {
        w.Write([]byte("admin: index"))
    })
    // ... more admin routes
    return r
}
```

## API Versioning

The versions example [`_examples/versions/main.go`](../../_examples/versions/main.go) demonstrates multiple API versions served from the same binary with version-specific presenters.

[`_examples/versions/main.go:29-46`](../../_examples/versions/main.go#L29-L46) mounts the same article router under `/v1`, `/v2`, and `/v3`, each with its own middleware:

```go
r.Route("/v1", func(r chi.Router) {
    r.Use(apiVersionCtx("v1"))
    r.Mount("/articles", articleRouter())
})
```

The `apiVersionCtx` middleware [`_examples/versions/main.go:51-58`](../../_examples/versions/main.go#L51-L58) stores the version string on the context. Handler logic branches on that value to select the appropriate presenter [`_examples/versions/main.go:84-92`](../../_examples/versions/main.go#L84-L92):

```go
apiVersion := r.Context().Value("api.version").(string)
switch apiVersion {
case "v1":
    articles <- v1.NewArticleResponse(article)
case "v2":
    articles <- v2.NewArticleResponse(article)
default:
    articles <- v3.NewArticleResponse(article)
}
```

Each presenter (v1, v2, v3) implements `render.Renderer` to transform the internal `Article` data model into version-specific JSON. [`_examples/versions/presenter/v3/article.go:25-35`](../../_examples/versions/presenter/v3/article.go#L25-L35) shows context-aware rendering: fields are conditionally included based on authentication state.

## Graceful Shutdown

[`_examples/graceful/main.go`](../../_examples/graceful/main.go) demonstrates production-ready server shutdown that allows in-flight requests to complete within a timeout.

[`_examples/graceful/main.go:21-23`](../../_examples/graceful/main.go#L21-L23) creates a context that listens for interrupt signals:

```go
ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
defer stop()
```

[`_examples/graceful/main.go:26-30`](../../_examples/graceful/main.go#L26-L30) runs the server in a background goroutine:

```go
go func() {
    if err := server.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
        log.Fatal(err)
    }
}()
```

[`_examples/graceful/main.go:33-42`](../../_examples/graceful/main.go#L33-L42) waits for the signal, then calls `server.Shutdown()` with a 30-second timeout:

```go
<-ctx.Done()

shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()

if err := server.Shutdown(shutdownCtx); err != nil {
    log.Fatal(err)
}
```

This ensures that running handlers [`_examples/graceful/main.go:55-66`](../../_examples/graceful/main.go#L55-L66) either complete or are interrupted when the timeout expires, preventing zombie goroutines.

## Static File Serving

[`_examples/fileserver/main.go:48-64`](../../_examples/fileserver/main.go#L48-L64) provides a reusable `FileServer` helper that safely mounts a `http.FileSystem` on a route. It validates the route pattern, handles trailing slashes, and uses `chi.RouteContext()` to extract the matched prefix for correct path stripping:

```go
func FileServer(r chi.Router, path string, root http.FileSystem) {
    if strings.ContainsAny(path, "{}*") {
        panic("FileServer does not permit any URL parameters.")
    }
    
    if path != "/" && path[len(path)-1] != '/' {
        r.Get(path, http.RedirectHandler(path+"/", 301).ServeHTTP)
        path += "/"
    }
    path += "*"
    
    r.Get(path, func(w http.ResponseWriter, r *http.Request) {
        rctx := chi.RouteContext(r.Context())
        pathPrefix := strings.TrimSuffix(rctx.RoutePattern(), "/*")
        fs := http.StripPrefix(pathPrefix, http.FileServer(root))
        fs.ServeHTTP(w, r)
    })
}
```

## Middleware Stack and Composition

[`_examples/rest/main.go:60-64`](../../_examples/rest/main.go#L60-L64) shows a typical middleware order for a REST service:

```go
r.Use(middleware.RequestID)
r.Use(middleware.Logger)
r.Use(middleware.Recoverer)
r.Use(middleware.URLFormat)
r.Use(render.SetContentType(render.ContentTypeJSON))
```

This ensures requests are identified, logged, panics are recovered, content negotiation works, and responses default to JSON. See [Middleware Fundamentals](middleware-fundamentals.md) for details on each.

Route-specific middleware is applied with `.With()` [`_examples/rest/main.go:80`](../../_examples/rest/main.go#L80) or `.Use()` within a subrouter [`_examples/rest/main.go:85`](../../_examples/rest/main.go#L85).

## Route Documentation

The rest example includes a flag to generate route documentation [[cite:_examples/rest/main.go:53, 99-109]]:

```bash
go run . -routes
```

This outputs a markdown file listing all routes, their middleware chains, and handlers. Use `docgen.JSONRoutesDoc()` or `docgen.MarkdownRoutesDoc()` from the chi repository to inspect your router structure programmatically.
