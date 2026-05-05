# Context — Propagation, Cancellation, Values

## Go Context is Mandatory

Every function that does I/O (DB, HTTP, file, goroutine) must accept `context.Context` as first parameter.

```go
func FetchUser(ctx context.Context, id uint) (*User, error) {
    var user User
    err := DB.WithContext(ctx).First(&user, id).Error
    if err != nil {
        return nil, fmt.Errorf("fetch user %d: %w", id, err)
    }
    return &user, nil
}
```

## Gin Context Propagation

```go
// Gin passes context through c.Request.Context()
func getUser(c *gin.Context) {
    user, err := FetchUser(c.Request.Context(), userID)
    if err != nil {
        c.JSON(500, gin.H{"error": err.Error()})
        return
    }
    c.JSON(200, user)
}
```

## When to Use Each Context Method

| Method | Use When |
|---|---|
| `context.Background()` | Entry point, tests, no request |
| `context.TODO()` |暂时未确定传递什么 context |
| `context.WithCancel()` | Cancel on another goroutine |
| `context.WithTimeout()` | HTTP client with timeout, DB queries |
| `context.WithValue()` | Request-scoped values (request ID, user) |
| `c.Request.Context()` | Gin handler → service layer |

## Context Cancellation

```go
func handler(c *gin.Context) {
    // Create a context that cancels if the HTTP request is cancelled
    ctx, cancel := context.WithTimeout(c.Request.Context(), 5*time.Second)
    defer cancel()

    result, err := slowOperation(ctx)
    if err != nil {
        if ctx.Err() == context.Canceled {
            c.JSON(499, gin.H{"error": "client cancelled"})
            return
        }
        if ctx.Err() == context.DeadlineExceeded {
            c.JSON(504, gin.H{"error": "operation timed out"})
            return
        }
        c.JSON(500, gin.H{"error": err.Error()})
        return
    }
    c.JSON(200, result)
}
```

## Context Values (Request Scoping)

```go
// Define type-safe keys to avoid collisions
type contextKey string
const requestIDKey contextKey = "request_id"
const userIDKey contextKey = "user_id"

// Set value
ctx := context.WithValue(ctx, requestIDKey, requestID)

// Get value
if requestID, ok := ctx.Value(requestIDKey).(string); ok {
    log.Info("request", "id", requestID)
}
```

**In Gin middleware:**
```go
func RequestMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        requestID := c.GetHeader("X-Request-ID")
        if requestID == "" {
            requestID = uuid.New().String()
        }
        
        // Store in Gin context AND propagate to service layer
        c.Set("request_id", requestID)
        
        // Create child context with request-scoped values
        ctx := context.WithValue(c.Request.Context(), requestIDKey, requestID)
        c.Request = c.Request.WithContext(ctx)
        
        c.Next()
    }
}
```

**In service layer:**
```go
func (s *UserService) GetUser(ctx context.Context, id uint) (*User, error) {
    requestID, _ := ctx.Value(requestIDKey).(string)
    log.Info("fetching user", "request_id", requestID, "user_id", id)
    
    // Use ctx for all DB operations
    return s.repo.FindByID(ctx, id)
}
```

## slog.Logger with Context (Go 1.21+)

Go 1.21+ `log/slog` supports structured logging with context built-in:

```go
import "log/slog"

func handlerWithSlog(c *gin.Context) {
    logger := slog.Default()
    
    // slog.Context returns a logger with ctx values attached
    // (requires Go 1.21+ with slog.FromContext, slog.WithContext)
    
    ctx := c.Request.Context()
    
    // Attach request-scoped values
    logger = logger.With(
        slog.String("request_id", c.GetString("request_id")),
        slog.String("client_ip", c.ClientIP()),
    )
    
    logger.InfoContext(ctx, "processing request",
        slog.String("method", c.Request.Method),
        slog.String("path", c.Request.URL.Path),
    )
    
    // slog automatically includes ctx values in Context-enabled handlers
    c.Next()
    
    logger.InfoContext(ctx, "request completed",
        slog.Int("status", c.Writer.Status()),
        slog.Duration("latency", time.Since(start)),
    )
}
```

** slog.Default() is package-level — for concurrent safety, use slog.New() with a handler:**

```go
import "log/slog"

var logger *slog.Logger

func init() {
    logger = slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelInfo,
    }))
}

// Use logger with context
func backgroundTask(ctx context.Context) {
    logger.InfoContext(ctx, "task started")
}
```

## Structured Error Response with Request ID

RFC 9457 (problem details) for standardized error responses:

```go
// RFC 9457 — Problem Details for HTTP APIs
type ProblemDetail struct {
    Type     string `json:"type"`
    Title    string `json:"title"`
    Status   int    `json:"status"`
    Detail   string `json:"detail,omitempty"`
    Instance string `json:"instance,omitempty"`
    RequestID string `json:"request_id,omitempty"`
}

func errorResponse(c *gin.Context, status int, title, detail string) {
    requestID, _ := c.Get("request_id")
    
    c.AbortWithStatusJSON(status, ProblemDetail{
        Type:     "about:blank",
        Title:    title,
        Status:   status,
        Detail:   detail,
        Instance: c.Request.URL.Path,
        RequestID: func() string {
            if r, ok := requestID.(string); ok {
                return r
            }
            return ""
        }(),
    })
}

// Usage
func handleError(c *gin.Context, err error) {
    if errors.Is(err, ErrNotFound) {
        errorResponse(c, 404, "Resource not found", err.Error())
        return
    }
    if errors.Is(err, ErrUnauthorized) {
        errorResponse(c, 401, "Unauthorized", "authentication required")
        return
    }
    errorResponse(c, 500, "Internal server error", "an unexpected error occurred")
}
```

## Context for Background Jobs

```go
// Long-running background work tied to request context
func backgroundUpload(c *gin.Context, file *multipart.FileHeader) error {
    ctx := c.Request.Context() // cancelled if client disconnects
    
    for chunk := range readChunks(file) {
        select {
        case <-ctx.Done():
            return ctx.Err() // client cancelled or request timed out
        default:
            if err := uploadChunk(ctx, chunk); err != nil {
                return err
            }
        }
    }
    return nil
}

// With deadline per operation
func batchProcess(ctx context.Context, items []Item) error {
    for i, item := range items {
        // Each item has 10 seconds max
        itemCtx, cancel := context.WithTimeout(ctx, 10*time.Second)
        defer cancel()
        
        if err := processItem(itemCtx, item); err != nil {
            log.Printf("item %d failed: %v", i, err)
            // Don't stop on single item failure — continue batch
        }
    }
    return nil
}
```

## Context for Multiple并发 Operations

```go
// Fan-out with shared context — any failure cancels all
func parallelFetch(ctx context.Context, userID string) (*User, []Post, error) {
    ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
    defer cancel()
    
    g, ctx := errgroup.WithContext(ctx)
    
    var user *User
    var posts []Post
    
    g.Go(func() error {
        var err error
        user, err = userRepo.GetByID(ctx, userID)
        return err
    })
    
    g.Go(func() error {
        var err error
        posts, err = postRepo.GetByUserID(ctx, userID, 10)
        return err
    })
    
    if err := g.Wait(); err != nil {
        return nil, nil, err // ctx is cancelled on first error
    }
    
    return user, posts, nil
}
```

## Context Values Gotchas

1. **Don't use `ctx.Value()` for mandatory data** — only use for optional, non-critical metadata
2. **Request ID is the most common value** — useful for log correlation across services
3. **Never store pointers in context values** — store IDs/primitive types instead
4. **Values don't cancel** — only `Done()` channel signals cancellation
5. **`context.WithValue` creates a new context** — original ctx is unchanged; you must pass the new ctx around

## Common Mistakes

1. **Passing `nil` context** — always pass a valid context, even if it's `context.Background()`
2. **Not using `WithContext`** — DB/GORM operations without `ctx` can't be cancelled
3. **Context values not type-safe** — use custom `contextKey` type, not plain strings
4. **Holding context too long** — cancel child contexts when work is done
5. **Values leaking between requests** — don't store request data in package-level variables
6. **Not checking `ctx.Done()` in loops** — long-running loops may never respect cancellation
7. **Storing large objects in context** — context values are copied; keep values small (IDs, strings)

---

## Updated from Research (2026-05)

### slog with Context (Go 1.21+)
- `slog.Logger` supports `InfoContext`, `ErrorContext` etc. for context-integrated logging
- Attach request ID, user ID, client IP as logger attributes for all subsequent log entries
- Use `slog.New()` with `slog.NewJSONHandler` for production structured logging

### RFC 9457 (Problem Details)
- Standardized error response format for HTTP APIs
- `type`, `title`, `status`, `detail`, `instance` fields
- Include `request_id` for traceability

### Distributed Tracing
- OpenTelemetry uses context values: `trace.ContextWithTraceID`, `trace.SpanFromContext`
- Attach trace ID as a context value for cross-service log correlation

### go.mod dependency for slog
- slog is in Go standard library since Go 1.21 — no external import needed
- Use `golang.org/x/sync/errgroup` for cleaner parallel error propagation

### Sources
- [Go Context docs](https://pkg.go.dev/context)
- [log/slog docs](https://pkg.go.dev/log/slog)
- [RFC 9457 — Problem Details](https://www.rfc-editor.org/rfc/rfc9457)