# Logging and Error Recovery

<!-- Maintained by Tidewiki. Edits are kept on later updates; wrap text in tidewiki:keep markers to freeze it. -->

Configure request logging and automatic panic recovery to handle errors gracefully in production services. Chi provides built-in middleware for both structured request logging and panic recovery with detailed stack traces.

## Request Logging

The `Logger` middleware [`middleware/logger.go:39-41`](../../middleware/logger.go#L39-L41) records the start and end of each request, including the HTTP method, path, response status, bytes written, and elapsed time. It automatically uses colored output when writing to a TTY, and monochrome output otherwise.

**Important:** Logger must be placed before `Recoverer` in your middleware chain [`middleware/logger.go:32-38`](../../middleware/logger.go#L32-L38) so that it logs even when a panic is recovered.

### Basic Usage

```go
import "github.com/go-chi/chi/v5"
import "github.com/go-chi/chi/v5/middleware"

r := chi.NewRouter()
r.Use(middleware.Logger)      // Must come before Recoverer
r.Use(middleware.Recoverer)
r.Get("/", func(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("Hello"))
})
```

The default logger prints to stdout with timestamps and colors:
```
15:04:05 "GET http://localhost:3000/ HTTP/1.1" from 127.0.0.1 - 200B in 123.456µs
```

If a request ID is available in the context (via `middleware.RequestID`), it will be included in the log output [`middleware/logger.go:109-112`](../../middleware/logger.go#L109-L112).

### Custom Log Formatting

Use `RequestLogger` with a custom `LogFormatter` to control log output:

```go
type LogEntry interface {
	Write(status, bytes int, header http.Header, elapsed time.Duration, extra interface{})
	Panic(v interface{}, stack []byte)
}

type LogFormatter interface {
	NewLogEntry(r *http.Request) LogEntry
}
```

[`middleware/logger.go:61-72`](../../middleware/logger.go#L61-L72)

Example with a custom formatter:

```go
type MyLogFormatter struct {
	Logger middleware.LoggerInterface
}

func (f *MyLogFormatter) NewLogEntry(r *http.Request) middleware.LogEntry {
	// Return a custom LogEntry implementation
	return &myLogEntry{r: r}
}

r := chi.NewRouter()
r.Use(middleware.RequestLogger(&MyLogFormatter{
	Logger: log.New(os.Stdout, "", log.LstdFlags),
}))
```

### Accessing the Log Entry

Inside a handler or downstream middleware, retrieve the current request's log entry using `GetLogEntry`:

```go
r.Get("/example", func(w http.ResponseWriter, r *http.Request) {
	entry := middleware.GetLogEntry(r)
	if entry != nil {
		// entry can be used to log additional data if implementing Write
	}
})
```

[`middleware/logger.go:74-78`](../../middleware/logger.go#L74-L78)

## Panic Recovery

The `Recoverer` middleware [`middleware/recoverer.go:22-49`](../../middleware/recoverer.go#L22-L49) recovers from panics, logs the panic and backtrace, and returns HTTP 500 (Internal Server Error) to the client. This prevents the server from crashing on unhandled panics in handlers.

### Basic Usage

```go
r := chi.NewRouter()
r.Use(middleware.Logger)      // Log before recovering
r.Use(middleware.Recoverer)   // Recover panics
r.Get("/crash", func(w http.ResponseWriter, r *http.Request) {
	panic("something went wrong")  // Will be caught and logged
})
```

When a panic occurs:
1. The panic is caught [`middleware/recoverer.go:24-30`](../../middleware/recoverer.go#L24-L30)
2. If a log entry exists, `Panic` is called on it with the recovered value and stack trace [`middleware/recoverer.go:32-34`](../../middleware/recoverer.go#L32-L34)
3. HTTP 500 is written to the response [`middleware/recoverer.go:39-41`](../../middleware/recoverer.go#L39-L41)
4. The stack trace is printed to stderr with color formatting [`middleware/recoverer.go:54-64`](../../middleware/recoverer.go#L54-L64)

### Special Case: http.ErrAbortHandler

The special panic value `http.ErrAbortHandler` is **not** recovered and will propagate normally [`middleware/recoverer.go:26-29`](../../middleware/recoverer.go#L26-L29). This is standard Go HTTP server behavior for intentional response aborts.

### Stack Trace Output

Panics are formatted with colored output highlighting the immediate panic location and the call stack:

```
 panic: something went wrong

 -> mypackage.panickingHandler
 ->   /path/to/file.go:42

    main.main
      /path/to/main.go:15

    runtime.main
      /path/to/runtime/proc.go:250
```

[`middleware/recoverer.go:54-109`](../../middleware/recoverer.go#L54-L109)

## Decisions

### Logger Placement Before Recoverer

Logger must be registered before Recoverer in the middleware chain. This ensures that request logging happens even when a panic is caught and recovered, providing complete visibility into requests that fail with panics [`middleware/logger.go:32-38`](../../middleware/logger.go#L32-L38).
