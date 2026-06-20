# Middleware — Logging, Recovery, CORS, Custom

## Built-in Middleware

```go
r := gin.New()

// Recovery — recovers from panics, returns 500
r.Use(gin.Recovery())

// Logger — logs requests with latency and status
r.Use(gin.Logger())

// Default includes both
r := gin.Default()
```

## Custom Middleware

```go
// Request ID middleware
func RequestID() gin.HandlerFunc {
    return func(c *gin.Context) {
        requestID := c.GetHeader("X-Request-ID")
        if requestID == "" {
            requestID = generateUUID()
        }
        c.Set("request_id", requestID)
        c.Header("X-Request-ID", requestID)
        c.Next()
    }
}

// Usage
r.Use(RequestID())
```

## Logging Middleware

```go
import "log/slog"

func Logger() gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()
        path := c.Request.URL.Path
        raw := c.Request.URL.RawQuery
        
        // Process request
        c.Next()
        
        // Log after request
        latency := time.Since(start)
        status := c.Writer.Status()
        clientIP := c.ClientIP()
        method := c.Request.Method
        
        if raw != "" {
            path = path + "?" + raw
        }
        
        slog.Info("request",
            "status", status,
            "method", method,
            "ip", clientIP,
            "path", path,
            "latency", latency,
        )
    }
}
```

**Skip Query String in Logs (Gin v1.12+):**
```go
import "github.com/gin-gonic/gin"

r.Use(gin.LoggerWithConfig(gin.LoggerConfig{
    SkipQueryString: true, // Omits ?foo=bar&secret=key from logs
}))
```
Useful for APIs where query strings may contain sensitive data (API keys, tokens).

**Pre-Go 1.21 (fallback):**
```go
import "log"


func Logger() gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()
        c.Next()
        log.Printf("[%d] %s %s %s %v",
            c.Writer.Status(),
            c.Request.Method,
            c.ClientIP(),
            c.Request.URL.Path,
            time.Since(start),
        )
    }
}
```

## Recovery with Custom Handler

```go
r.Use(gin.CustomRecovery(func(c *gin.Context, err interface{}) {
    log.Printf("Panic recovered: %v", err)
    c.AbortWithStatusJSON(500, gin.H{
        "error": "internal server error",
        "request_id": c.GetString("request_id"),
    })
}))
```

## CORS Middleware

```go
import "github.com/gin-contrib/cors"

// Option 1: Default (all origins, specific methods)
r.Use(cors.Default())

// Option 2: Custom config
r.Use(cors.New(cors.Config{
    AllowOrigins:     []string{"https://myapp.com"},
    AllowMethods:     []string{"GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"},
    AllowHeaders:     []string{"Origin", "Content-Type", "Authorization", "X-Request-ID"},
    ExposeHeaders:    []string{"X-RateLimit-Remaining"},
    AllowCredentials: true,
    MaxAge:           12 * time.Hour,
}))
```

## Rate Limiting Middleware

```go
import (
    "net/http"
    "sync"
    "time"
    "golang.org/x/time/rate"
)

// IPRateLimiter — simple per-IP rate limiter
type IPRateLimiter struct {
    ips    map[string]*rate.Limiter
    mu     sync.RWMutex
    rate   rate.Limit
    burst  int
}

func NewIPRateLimiter(r rate.Limit, b int) *IPRateLimiter {
    return &IPRateLimiter{
        ips:   make(map[string]*rate.Limiter),
        rate:  r,
        burst: b,
    }
}

func (i *IPRateLimiter) GetLimiter(ip string) *rate.Limiter {
    i.mu.Lock()
    defer i.mu.Unlock()
    
    limiter, exists := i.ips[ip]
    if !exists {
        limiter = rate.NewLimiter(i.rate, i.burst)
        i.ips[ip] = limiter
    }
    
    return limiter
}

func RateLimitMiddleware(limiter *IPRateLimiter) gin.HandlerFunc {
    return func(c *gin.Context) {
        ip := c.ClientIP()
        if !limiter.GetLimiter(ip).Allow() {
            c.AbortWithStatusJSON(http.StatusTooManyRequests, gin.H{
                "error": "rate limit exceeded",
            })
            return
        }
        c.Next()
    }
}

// Usage
r := gin.Default()
limiter := NewIPRateLimiter(rate.Limit(10), 20) // 10 req/s, burst 20
r.Use(RateLimitMiddleware(limiter))
```

## Middleware with Context

```go
// Middleware that calls service layer
func DatabaseMiddleware(db *gorm.DB) gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Set("db", db)
        c.Next()
    }
}

func getDB(c *gin.Context) *gorm.DB {
    return c.MustGet("db").(*gorm.DB)
}
```

## Middleware Execution Order

```go
r.Use(m1) // 1st
r.Use(m2) // 2nd
r.GET("/path", h) // handler

// Execution order: m1 → m2 → h → m2 (response) → m1 (response)
```

## Timeout Middleware with errgroup (Recommended)

The naive goroutine + channel timeout pattern cannot propagate errors from the request handler back to the middleware context. Use `errgroup` with a derived context for proper error propagation:

```go
import (
    "context"
    "errors"
    "net/http"
    "time"

    "github.com/gin-gonic/gin"
    "golang.org/x/sync/errgroup"
)

// ErrTimeout is returned when a request exceeds the timeout limit
var ErrTimeout = errors.New("request timeout")

// TimeoutMiddleware returns a middleware that timeout-halts the request
// after the given duration. Errors from the handler are propagated correctly.
func TimeoutMiddleware(timeout time.Duration) gin.HandlerFunc {
    return func(c *gin.Context) {
        // Create a context that cancels on timeout
        ctx, cancel := context.WithTimeout(c.Request.Context(), timeout)
        defer cancel()

        // Replace request context so downstream handlers inherit the timeout
        c.Request = c.Request.WithContext(ctx)

        eg, ctx := errgroup.WithContext(ctx)

        eg.Go(func() error {
            c.Next()
            // c.Errors contains any errors set by handlers (abort, validation, etc.)
            if len(c.Errors) > 0 {
                // Return the first non-nil error, or a sentinel if handlers called c.Abort
                for _, e := range c.Errors {
                    if e.Err != nil {
                        return e.Err
                    }
                }
                // c.Abort was called without setting an error
                return c.Aborted()
            }
            return nil
        })

        // Wait for handler + timeout race
        if err := eg.Wait(); err != nil {
            if errors.Is(err, context.DeadlineExceeded) {
                c.AbortWithStatusJSON(http.StatusGatewayTimeout, gin.H{
                    "error": "request timeout",
                })
                return
            }
            // Other error from handler — don't overwrite if already written
            if !c.Writer.Written() {
                c.AbortWithStatusJSON(http.StatusInternalServerError, gin.H{
                    "error": "request failed",
                })
            }
        }
    }
}

// c.Aborted() returns true if c.Abort* was called
// Use it to distinguish aborted requests from panics
```

**Why errgroup over channel + goroutine:**
- Propagates handler errors (panic, abort, validation failures) back to the middleware
- Properly cancels the context when the timeout fires, causing in-flight DB calls etc. to abort
- `errgroup.WithContext` automatically cancels the group context if any goroutine returns a non-nil error

## Common Mistakes

1. **`c.Next()` not called** — middleware chain breaks, downstream handlers never run
2. **Aborting without status** — `c.Abort()` returns empty 0 status, use `c.AbortWithStatusJSON()`
3. **Goroutine in middleware without context** — async work loses request context
4. **Rate limiter memory leak** — clean up old entries periodically
5. **Middleware modifying response** — after `c.Next()`, response headers are already sent
6. **Naive timeout using channels** — can't propagate handler errors; use `errgroup` instead

## Goroutine Leak Detection (Go 1.27+)

Go 1.27 graduates the **goroutine leak profiler** to GA. Expose it on your Gin server
to catch leaked goroutines in production (or staging) — extremely useful for catching
async middleware bugs that would otherwise only show up as memory creep.

```go
import (
    "net/http"
    "net/http/pprof"

    "github.com/gin-gonic/gin"
)

// Expose /debug/pprof endpoints (including the new goroutineleak profile)
func RegisterPprof(r *gin.Engine) {
    debug := r.Group("/debug/pprof")
    // Catch-all for the index page
    debug.GET("/*any", gin.WrapH(http.HandlerFunc(pprof.Index)))
    debug.POST("/*any", gin.WrapH(http.HandlerFunc(pprof.Index)))

    // Named profiles that aren't covered by the catch-all
    profiles := map[string]http.HandlerFunc{
        "cmdline": pprof.Cmdline,
        "profile": pprof.Profile,
        "symbol":  pprof.Symbol,
        "trace":   pprof.Trace,
    }
    for name, h := range profiles {
        debug.GET("/"+name, gin.WrapH(h))
    }

    // The new endpoint (Go 1.27 GA): GET /debug/pprof/goroutineleak
    debug.GET("/goroutineleak", gin.WrapH(pprof.Handler("goroutineleak")))
}
```

**Production usage:**
- Hit `GET /debug/pprof/goroutineleak` periodically (cron, k8s probe, or on alert)
- The output lists goroutines that *should* have exited but are still running
- Common offenders: background workers spawned in middleware that never receive a
  shutdown signal, `c.Next()` goroutines that lost their context cancellation

**Note:** Requires Go 1.27+. In Go 1.26 the same endpoint exists behind
`GOEXPERIMENT=goroutineleakprofile` at build time.

See [versions.md](versions.md) for the full Go 1.27 feature list.
