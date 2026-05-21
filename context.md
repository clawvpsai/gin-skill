# Context — Gin Context vs Go Context, Propagation, Cancellation

## Critical Distinction: `*gin.Context` vs `context.Context`

**These are two completely different types:**

| Type | Package | What it is |
|---|---|---|
| `*gin.Context` | `github.com/gin-gonic/gin` | Gin-specific — holds request, response, params, Gin-specific methods |
| `context.Context` | `context` (stdlib) | Standard library — cancellation, deadlines, request-scoped values |

**`gin.Context` embeds `*http.Request` which has a `Context()` method:**

```go
// Get Go context from Gin context
ctx := c.Request.Context()

// NOT the same as c.Context (which is Gin-specific)
```

## Extracting Go Context from Gin

```go
func handler(c *gin.Context) {
    // c.Request.Context() — the standard Go context
    ctx := c.Request.Context()

    // c.Context — Gin's internal context (stores Gin-specific values)
    // Only use c.Context for Gin-specific operations, not standard context operations
}
```

**Never create a new `context.Background()` in a Gin handler** — use the request context:

```go
// WRONG — loses cancellation, deadlines, request-scoped values
ctx := context.Background()

// RIGHT — inherits from the HTTP request
ctx := c.Request.Context()
```

## Setting Request-Scoped Values

Use `c.Request.Context()` with `WithValue` for request-scoped data that needs to propagate to lower layers (DB, HTTP clients):

```go
func authMiddleware(c *gin.Context) {
    ctx := c.Request.Context()

    // Store user ID in context for propagation to DB/service layers
    ctx = context.WithValue(ctx, "user_id", userID)

    // Replace the request's context (propagates to all I/O operations)
    c.Request = c.Request.WithContext(ctx)

    c.Next()
}

func handler(c *gin.Context) {
    ctx := c.Request.Context()

    // Retrieve from Go context (not Gin c.Get)
    userID := ctx.Value("user_id")
    if userID == nil {
        c.JSON(401, gin.H{"error": "unauthorized"})
        return
    }

    user, err := db.GetUserByID(ctx, userID.(int))
    // ctx is passed to DB — cancellation propagates automatically
}
```

**For Gin-specific values (params, headers, etc.), use Gin context:**

```go
// Gin values — for Gin layer only
c.Set("request_id", "abc")
requestID := c.GetString("request_id")

// Go context values — for service/DB layer propagation
ctx := context.WithValue(c.Request.Context(), "request_id", "abc")
value := ctx.Value("request_id")
```

**Why the distinction matters:**
- `c.Set`/`c.Get` are Gin-specific — useful for middleware-to-handler communication within Gin
- `ctx.Value` propagates through `db.Query(ctx, ...)`, `httpClient.Do(ctx, ...)`, etc. — works across package boundaries

## Context Cancellation

```go
func handler(c *gin.Context) {
    ctx := c.Request.Context()

    // Check if context was cancelled (client disconnected)
    select {
    case <-ctx.Done():
        // Client disconnected — don't start new work
        return
    default:
        // Continue
    }

    // Context cancellation propagates automatically to:
    // - DB queries
    // - HTTP client calls
    // - File operations
    // - Any operation that accepts context
}
```

**Common cancellation patterns:**

```go
// Timeout for an operation
func handler(c *gin.Context) {
    ctx, cancel := context.WithTimeout(c.Request.Context(), 5*time.Second)
    defer cancel()

    result, err := fetchData(ctx, url)
    if err != nil {
        if ctx.Err() == context.DeadlineExceeded {
            c.JSON(504, gin.H{"error": "operation timed out"})
            return
        }
        c.JSON(500, gin.H{"error": err.Error()})
        return
    }

    c.JSON(200, result)
}

// Deadline — absolute time
func handler(c *gin.Context) {
    ctx, cancel := context.WithDeadline(
        c.Request.Context(),
        time.Now().Add(10*time.Second),
    )
    defer cancel()

    // Same pattern as timeout
    result, err := fetchData(ctx, url)
    // ...
}

// WithCancelCause — attach an error to cancellation (Go 1.21+)
func fanOut(c *gin.Context) {
    ctx, cancel := context.WithCancelCause(c.Request.Context())
    defer cancel(context.Cause(ctx))

    g, ctx := errgroup.WithContext(ctx)
    for _, item := range items {
        g.Go(func() error {
            if err := process(item); err != nil {
                cancel(fmt.Errorf("item %v failed: %w", item, err))
                return err
            }
            return nil
        })
    }

    if err := g.Wait(); err != nil {
        log.Printf("failed: %v", context.Cause(ctx))
    }
}

## context.AfterFunc — Schedule Work on Context Cancellation (Go 1.21+)

`context.AfterFunc` schedules a function to run when a context is cancelled — without creating a new goroutine yourself:

```go
import "context"

// Schedule cleanup to run when request context is cancelled
func withCleanup(ctx context.Context, resource *Resource) {
    ctx, cancel := context.WithCancelCause(ctx)
    defer cancel(context.Cause(ctx)) // propagate original cause on return

    done := context.AfterFunc(ctx, func() {
        // Runs exactly once — either on ctx cancellation OR when WithCancelCause returns
        resource.Close()
        log.Printf("resource cleaned up: %v", context.Cause(ctx))
    })

    // If AfterFunc already fired (ctx cancelled before this line), done is false
    // If AfterFunc scheduled the function, done is true — don't call the cleanup again
    if !done() {
        // ctx was already cancelled before AfterFunc was called — cleanup already queued
        return
    }

    // Normal operation: ctx is still active, AfterFunc is registered
    // do work...
}

// Better pattern: always derive from the request context in Gin handlers
func handler(c *gin.Context) {
    reqCtx := c.Request.Context()

    // Schedule cleanup — runs when client disconnects or handler times out
    var resource Resource // imagine this holds a DB tx, file handle, etc.
    done := context.AfterFunc(reqCtx, func() {
        resource.Release()
    })
    defer func() {
        if done() {
            // ctx still active — cancel to trigger the AfterFunc
            // Use cancelCause to attach an error to the cancellation
        }
    }()

    // ... handler work ...
}
```

**Why use `AfterFunc` over a manual goroutine:**
- No goroutine leak risk — the function runs exactly once, guaranteed
- No `sync.Once` needed — the function is de-duplicated automatically
- Works correctly with context cancellation AND `defer cancel()` on normal return
- Useful for cleanup: closing DB transactions, releasing file handles, cancelling background work

**Key behavior:**
- Returns a `func() bool` — call it to check if the function was already scheduled to run
- If `done()` returns `false`, ctx was already cancelled before `AfterFunc` was called — the function already ran or will run shortly
- If `done()` returns `true`, the function is registered and will run when ctx is cancelled

```

## Context with Database Operations

```go
// Pass context to GORM — enables cancellation, timeouts, tracing
func getUser(ctx context.Context, id int) (*User, error) {
    var user User
    // GORM automatically uses ctx for cancellation
    if err := db.WithContext(ctx).First(&user, id).Error; err != nil {
        return nil, err
    }
    return &user, nil
}

// In Gin handler — always pass request context
func getUserHandler(c *gin.Context) {
    var param struct {
        ID int `uri:"id" binding:"required"`
    }
    if err := c.ShouldBindUri(&param); err != nil {
        c.JSON(400, gin.H{"error": "invalid id"})
        return
    }

    user, err := getUser(c.Request.Context(), param.ID)
    if err != nil {
        if err == gorm.ErrRecordNotFound {
            c.JSON(404, gin.H{"error": "user not found"})
            return
        }
        c.JSON(500, gin.H{"error": "failed to fetch user"})
        return
    }

    c.JSON(200, user)
}
```

**⚠️ Always pass context to GORM operations:**

```go
// WRONG — no cancellation propagation, long queries block forever
db.First(&user, id)

// RIGHT — cancellation propagates, DB operation respects context deadline
db.WithContext(ctx).First(&user, id)
```

## Context with HTTP Clients

```go
func callDownstreamService(ctx context.Context, url string) ([]byte, error) {
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return nil, err
    }

    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    return io.ReadAll(resp.Body)
}

// In Gin handler
func handler(c *gin.Context) {
    data, err := callDownstreamService(c.Request.Context(), "https://api.example.com/data")
    // If client disconnects, the HTTP request is cancelled automatically
}
```

## Context in Gin Middleware

```go
// Middleware that adds a value to request context
func tracingMiddleware(c *gin.Context) {
    ctx := c.Request.Context()

    // Add trace ID to context for propagation
    traceID := c.GetHeader("X-Trace-ID")
    if traceID == "" {
        traceID = generateUUID()
    }
    ctx = context.WithValue(ctx, "trace_id", traceID)
    c.Request = c.Request.WithContext(ctx)

    // Also set in Gin context for use within Gin handlers
    c.Set("trace_id", traceID)
    c.Header("X-Trace-ID", traceID)

    c.Next()
}

// Access in handler
func handler(c *gin.Context) {
    // Via Gin context (for Gin-layer operations)
    traceID := c.GetString("trace_id")

    // Via Go context (for service/DB layer propagation)
    traceID = c.Request.Context().Value("trace_id").(string)
}
```

## Using `c.Copy()` for Async Work

**When you need Gin context after the request handler returns (async work):**

```go
func asyncHandler(c *gin.Context) {
    // Copy the Gin context BEFORE starting goroutine
    // c.Copy() creates a copy with the same request and response writer
    ctxCopy := c.Copy()

    go func(cc *gin.Context) {
        // Use cc (the copy) — NOT c (original is invalidated after request)
        // Original c is only valid for the duration of the HTTP request

        time.Sleep(2 * time.Second)

        // cc.Writer still works for writing responses
        // (though for async, you'd typically write to DB, queue, etc.)
        log.Printf("async work done for request: %s", cc.Request.URL.Path)
    }(ctxCopy)

    // Return immediately — don't wait for goroutine
    c.JSON(202, gin.H{"message": "accepted"})
}
```

**Better pattern — use Go context for async work:**

```go
func betterAsyncHandler(c *gin.Context) {
    ctx := c.Request.Context()

    go func(ctx context.Context) {
        // Use Go context — survives the request
        // Cancellation of ctx signals when to stop

        select {
        case <-time.After(2 * time.Second):
            log.Printf("async work done")
        case <-ctx.Done():
            log.Printf("async work cancelled: %v", ctx.Err())
        }
    }(ctx) // ctx is cancelled when client disconnects or handler returns

    c.JSON(202, gin.H{"message": "accepted"})
}
```

## Context Values and Type Safety

Context values are untyped (`interface{}`). Use a custom key type to avoid collisions:

```go
// Private struct type — prevents collisions in context map
type contextKey string

const (
    userIDKey contextKey = "user_id"
    traceIDKey contextKey = "trace_id"
)

// Set value
ctx := context.WithValue(c.Request.Context(), userIDKey, 123)

// Get value (type-safe)
func getUserID(ctx context.Context) (int, bool) {
    val := ctx.Value(userIDKey)
    if val == nil {
        return 0, false
    }
    id, ok := val.(int)
    return id, ok
}

// In handler
func handler(c *gin.Context) {
    ctx := c.Request.Context()
    if userID, ok := getUserID(ctx); ok {
        log.Printf("user_id: %d", userID)
    }
}
```

**Why use unexported key types:**
```go
// WRONG — string key collides with any other package using "user_id"
ctx = context.WithValue(ctx, "user_id", 123)

// RIGHT — only your package can access this key
type key string
const userIDKey key = "user_id"
```

## Context with OpenTelemetry Tracing

OpenTelemetry uses context for trace propagation:

```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/trace"
)

// Extract trace context from incoming request
func tracingMiddleware(c *gin.Context) {
    tracer := otel.Tracer("my-service")

    // Start a new span with the incoming trace context
    ctx, span := tracer.Start(c.Request.Context(), c.FullPath())
    defer span.End()

    // Replace request context with trace-enabled context
    c.Request = c.Request.WithContext(ctx)

    c.Next()
}

// Propagate trace context to outgoing requests
func callService(ctx context.Context, url string) error {
    tracer := otel.Tracer("my-service")
    ctx, span := tracer.Start(ctx, "callService")
    defer span.End()

    req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)

    // Trace context headers are injected automatically
    // by OpenTelemetry's HTTP instrumentation
    resp, err := http.DefaultClient.Do(req)
    return err
}
```

## Common Mistakes

1. **`c.Context` vs `c.Request.Context()`** — `c.Context` is Gin-internal, `c.Request.Context()` is standard Go context
2. **`context.Background()` in handlers** — loses cancellation from client disconnect, timeouts don't work
3. **Not passing context to DB** — GORM queries ignore cancellation without `WithContext(ctx)`
4. **Goroutine using original `c`** — after handler returns, `c` is invalid; use `c.Copy()` or pass Go context
5. **String keys in context values** — collisions; use unexported custom key types
6. **Storing Gin-specific values in Go context** — `c.Set("user", user)` is fine within Gin; don't use `ctx.WithValue(ctx, "user", user)` for Gin values
7. **`context.WithValue` for secrets** — context values are not secure; use function parameters for sensitive data
8. **Not checking `ctx.Done()` in long operations** — waste resources after client disconnected

---

## Updated from Research (2026-05)

### context.AfterFunc (Go 1.21+)
- `context.AfterFunc(ctx, fn)` schedules `fn` to run exactly once when `ctx` is cancelled
- Returns `func() bool` — call it to check if the function was already scheduled
- No goroutine leak risk — the function runs exactly once, de-duplicated automatically
- Ideal for cleanup: closing DB transactions, releasing file handles, cancelling background work
- Prefer over manual goroutine + `sync.Once` patterns

### Gin Context vs Go Context
- `*gin.Context` is request-scoped and tied to the HTTP request lifecycle
- `*gin.Context` embeds `*http.Request`, which carries a `context.Context`
- Always pass `c.Request.Context()` to lower layers (DB, HTTP, Redis) — never `context.Background()`

### Context Propagation
- Use `c.Request = c.Request.WithContext(ctx)` to propagate context changes to downstream middleware
- For async work, prefer passing Go `context.Context` over using `c.Copy()`
- Context values use type-safe unexported key types to prevent collisions

### OpenTelemetry Integration
- Replace `c.Request.Context()` with a trace-instrumented context via `otel.Tracer().Start()`
- OpenTelemetry propagates trace context automatically through HTTP headers

### Sources
- https://gin-gonic.com/docs/context/ (Gin context methods)
- https://go.dev/blog/context (Go context blog post — still the definitive guide)
- https://pkg.go.dev/context (stdlib context package)
- https://opentelemetry.io/docs/languages/go/ (OpenTelemetry Go)
