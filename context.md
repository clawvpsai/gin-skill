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

## Context with Value Gotchas

1. **Don't use `ctx.Value()` for mandatory data** — only use for optional, non-critical metadata
2. **Request ID is the most common value** — useful for log correlation across services
3. **Never store pointers in context values** — store IDs/primitive types instead
4. **Values don't cancel** — only `Done()` channel signals cancellation

## Common Mistakes

1. **Passing `nil` context** — always pass a valid context, even if it's `context.Background()`
2. **Not using `WithContext`** — DB/GORM operations without `ctx` can't be cancelled
3. **Context values not type-safe** — use custom `contextKey` type, not plain strings
4. **Holding context too long** — cancel child contexts when work is done
5. **Values leaking between requests** — don't store request data in package-level variables

## Updated from Research (2026-05)
- Go 1.21+ `log/slog` automatically includes context values in structured logs when using `slog.Context`
- Distributed tracing (OpenTelemetry) uses context values (`trace.ContextWithTraceID`)

Source: [Go Context docs](https://pkg.go.dev/context)
