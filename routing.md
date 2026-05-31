# Routing — Router, Groups, Path Parameters

## Basic Router Setup

```go
package main

import (
    "github.com/gin-gonic/gin"
)

func main() {
    r := gin.Default() // includes Logger + Recovery middleware
    // Or for production:
    // r := gin.New()
    // r.Use(gin.Recovery(), gin.Logger())

    r.Run(":8080") // listens on :8080
}
```

## Gin v1.12 Escaped Path Option

```go
// Gin v1.12+ — use raw/escaped request path for routing
// Useful when dealing with URLs with encoded characters
r := gin.New()
r.UseEscapedPath = true // use url.EscapedPath() for routing
```

## Route Methods

```go
r.GET("/posts", getPosts)
r.POST("/posts", createPost)
r.PUT("/posts/:id", updatePost)
r.DELETE("/posts/:id", deletePost)
r.PATCH("/posts/:id", patchPost)
r.HEAD("/posts", headPosts)
r.OPTIONS("/posts", optionsPosts)
```

## Path Parameters

```go
// /posts/:id — required
// /posts/:id/comments/:comment_id — nested
r.GET("/posts/:id", func(c *gin.Context) {
    id := c.Param("id")

    // Multiple params
    commentID := c.Param("comment_id")

    c.JSON(200, gin.H{"id": id, "comment_id": commentID})
})

// /posts/*action — wildcard (captures everything after /posts/)
r.GET("/posts/*action", func(c *gin.Context) {
    action := c.Param("action") // "/posts/edit" → "edit"
})
```

**Route order matters — first match wins, put specific routes BEFORE generic ones:**
```go
// WRONG — /users/:id matches this first, "admin" is never reached
r.GET("/users/:id", getUser)
r.GET("/users/admin", getAdmin)

// RIGHT — specific routes first
r.GET("/users/admin", getAdmin)
r.GET("/users/:id", getUser)
```

## Query Parameters

```go
// GET /search?q=golang&page=1
r.GET("/search", func(c *gin.Context) {
    q := c.Query("q")                  // "golang" — returns "" if missing
    page := c.DefaultQuery("page", "1") // "1" if "page" not provided

    // Parse integer with fallback
    pageNum := 1
    if p, err := strconv.Atoi(c.Query("page")); err == nil && p > 0 {
        pageNum = p
    }

    // Check if param exists
    if c.Query("debug") != "" {
        // debug was provided
    }
})
```

## Router Groups

```go
// Group with shared prefix and middleware
api := r.Group("/api/v1")
api.Use(AuthMiddleware())        // applies to all routes in group
api.Use(LogMiddleware(), RateLimitMiddleware()) // multiple middleware

{
    api.GET("/posts", getPosts)
    api.POST("/posts", createPost)
    api.GET("/users", getUsers)
    api.PUT("/users/:id", updateUser)
}

// Nested groups
v1 := r.Group("/v1")
{
    v1.POST("/login", login)

    posts := v1.Group("/posts")
    {
        posts.GET("/", listPosts)
        posts.POST("/", createPost)
        posts.GET("/:id", getPost)
    }
}
```

## Matching All Methods

```go
r.Any("/unknown", func(c *gin.Context) {
    c.JSON(404, gin.H{"error": "not found"})
})
```

## Static Files

```go
import "net/http"

// Serve static files from a directory
r.Static("/assets", "./public")

// Serve a single file
r.StaticFile("/favicon.ico", "./public/favicon.ico")

// Alternative: using http.FileServer directly (more control)
r.GET("/files/*filepath", func(c *gin.Context) {
    filepath := c.Param("filepath")
    http.ServeFile(c.Writer, c.Request, "./public"+filepath)
})
```

## HTTP/3 Support (Gin v1.11+)

HTTP/3 support in Gin uses the `quic-go` library. Here's a practical setup:

```go
import (
    "context"
    "crypto/tls"
    "fmt"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"

    "github.com/gin-gonic/gin"
    "github.com/quic-go/quic-go/http3"
)
```

**Option 1: Standalone HTTP/3 server (no HTTP/1.1 on the same port)**

QUIC/UDP requires a separate port from HTTP/1.1. Gin serves HTTP/3 on UDP:

```go
func runHTTP3(r *gin.Engine, addr string) error {
    server := &http3.Server{
        Addr:    addr,        // e.g., ":8443" (UDP)
        Handler: r,
        TLSConfig: &tls.Config{
            GetCertificate: func(hello *tls.ClientHelloInfo) (*tls.Certificate, error) {
                // Load your wildcard cert, or use certmagic/autocert
                return loadCertificate()
            },
        },
    }
    return server.ListenAndServe()
}
```

**Option 2: Dual-server setup — HTTP/1.1 + HTTP/3 on different ports**

The most reliable production pattern: HTTP/1.1 on one port, HTTP/3 on another. Typically:
- Port 8080: HTTP/1.1 (HTTP/2 via ALPN upgrade)
- Port 8443: HTTP/3 over QUIC/UDP

```go
func main() {
    r := gin.Default()
    // ... register routes ...

    // HTTP/3 on UDP :8443
    http3Server := &http3.Server{
        Addr:      ":8443",
        Handler:   r,
        TLSConfig: tlsConfig(),
    }

    // HTTP/1.1/HTTP/2 on :8080
    httpServer := &http.Server{
        Addr:    ":8080",
        Handler: r,
    }

    // Start HTTP/3 in a goroutine
    go func() {
        fmt.Println("HTTP/3 server listening on :8443 (UDP)")
        if err := http3Server.ListenAndServe(); err != nil {
            fmt.Printf("HTTP/3 server error: %v\n", err)
        }
    }()

    // Graceful shutdown
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    http3Server.Shutdown(ctx)
    httpServer.Shutdown(ctx)
}

func tlsConfig() *tls.Config {
    certFile := os.Getenv("CERT_FILE")   // fullchain.pem
    keyFile := os.Getenv("KEY_FILE")     // privkey.pem
    cert, err := tls.LoadX509KeyPair(certFile, keyFile)
    if err != nil {
        panic("failed to load TLS certificate: " + err.Error())
    }
    return &tls.Config{
        Certificates: []tls.Certificate{cert},
        // Enable HTTP/3 via ALPN — clients that support HTTP/3 will upgrade
        NextProtos: []string{"h3", "h2", "http/1.1"},
    }
}
```

**Option 3: Nginx in front with HTTP/3 (recommended for production)**

In most production setups, Nginx handles HTTP/3 termination and forwards HTTP/1.1 to Gin:

```nginx
# Nginx 1.25+ with HTTP/3 on port 443 (UDP + TCP)
server {
    listen 443 ssl http2;
    listen 443 http3 reuseport;  # UDP — enables HTTP/3

    ssl_certificate     /etc/letsencrypt/live/api.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.example.com/privkey.pem;

    # HTTP/3 requires TLS 1.3
    ssl_protocols TLSv1.3;
    quic_retry on;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

**quic-go version compatibility:**
- **quic-go v0.59.1** (stable, 2026-05-11) — current recommended for production
- **quic-go v0.60 (development)** — adds FIPS 140-3 compliance support when built with Go 1.26+; not yet stable
- Gin v1.12.x works with either version via `github.com/quic-go/quic-go/http3`
- Add to `go.mod`: `github.com/quic-go/quic-go v0.59.1`

**When to use HTTP/3:**
- Public APIs with diverse client networks (mobile, international)
- Real-time applications (WebSocket alternatives for browsers)
- When behind a HTTP/3-capable reverse proxy (Nginx 1.25+, Caddy)
- High-latency or unreliable network paths where QUIC's packet loss isolation helps

**When to skip:**
- Internal services behind a load balancer that already handles HTTP/2
- Latency-insensitive local networks
- When `quic-go` dependency complexity isn't worth the gain
- Environments where UDP traffic is blocked (some corporate networks)

## Common Mistakes

1. **Route order wrong** — generic `/:id` routes above specific `/admin` routes
2. **Not checking `c.Params` is empty** — param not found returns empty string
3. **Using wildcard `*` incorrectly** — `*path` captures including leading slash
4. **No 404 handler** — missing routes return 404 by default but can be customized
5. **Path params with same names in nested routes** — use unique names or `c.Params.ByName("id")`
6. **Not using escaped path routing when URL encoding matters** — set `r.UseEscapedPath = true` on the engine in Gin v1.12+
7. **Using `c.QueryInt()`** — doesn't exist in Gin, use `strconv.Atoi(c.Query("page"))` instead
8. **Static file setup without `net/http` import** — `http.ServeFile` needs the import
9. **Trying to serve HTTP/1.1 and HTTP/3 on the same TCP port** — QUIC runs on UDP; use separate ports
10. **HTTP/3 without TLS 1.3** — QUIC requires TLS 1.3 minimum; set `ssl_protocols TLSv1.3` in Nginx

---

## Updated from Research (2026-05-31)

### HTTP/3 in Production
- **Dual-server pattern** (HTTP/1.1 on TCP + HTTP/3 on UDP) is the standard Gin + quic-go setup
- **Nginx 1.25+** can terminate HTTP/3 and proxy HTTP/1.1 to Gin — this is the recommended production approach
- `quic_retry on` in Nginx enables connection migration (a key QUIC feature)
- QUIC requires **TLS 1.3** minimum — older TLS versions won't work with HTTP/3
- `ssl_protocols TLSv1.3` in Nginx is required for HTTP/3 to function

### quic-go v0.60 (dev)
- **quic-go v0.60 (development)** adds FIPS 140-3 compliance support when built with Go 1.26+
- For production, stick with **quic-go v0.59.1** (stable)
- The v0.60 dev version is useful for FIPS-compliant government/defense deployments

### Sources
- https://github.com/quic-go/quic-go/releases
- https://github.com/gin-gonic/gin/issues (HTTP/3 tracking)
- Nginx QUIC: https://nginx.org/en/docs/quic.html
