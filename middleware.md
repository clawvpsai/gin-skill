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
        
        log.Printf("[%d] %s %s %s %v", 
            status, method, clientIP, path, latency)
    }
}
```

## Recovery with Custom Handler

```go
import "github.com/gin-contrib/expvar"

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

## Timeout Middleware

```go
import "context"

func TimeoutMiddleware(timeout time.Duration) gin.HandlerFunc {
    return func(c *gin.Context) {
        ctx, cancel := context.WithTimeout(c.Request.Context(), timeout)
        defer cancel()
        
        c.Request = c.Request.WithContext(ctx)
        
        done := make(chan struct{})
        
        go func() {
            c.Next()
            close(done)
        }()
        
        select {
        case <-done:
            // Request completed normally
        case <-ctx.Done():
            c.AbortWithStatusJSON(504, gin.H{"error": "request timeout"})
        }
    }
}
```

## Common Mistakes

1. **`c.Next()` not called** — middleware chain breaks, downstream handlers never run
2. **Aborting without status** — `c.Abort()` returns empty 0 status, use `c.AbortWithStatusJSON()`
3. **Goroutine in middleware without context** — async work loses request context
4. **Rate limiter memory leak** — clean up old entries periodically
5. **Middleware modifying response** — after `c.Next()`, response headers are already sent