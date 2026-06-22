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
- **quic-go v0.59.1** (stable, 2026-05-11) — previous stable
- **quic-go v0.60.0** (stable, 2026-06-06) — adds FIPS 140-3 compliance support when built with Go 1.26+; Go 1.25+ required
- Gin v1.12.x works with either version via `github.com/quic-go/quic-go/http3`
- Add to `go.mod`: `github.com/quic-go/quic-go v0.60.0`

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

### quic-go v0.60.0 (stable)
- **quic-go v0.60.0** adds FIPS 140-3 compliance support; Go 1.25+ required
- For production, use **quic-go v0.60.0** (stable, 2026-06-06)
- v0.59.1 still works but lacks FIPS 140-3 support

### Sources
- https://github.com/quic-go/quic-go/releases
- https://github.com/gin-gonic/gin/issues (HTTP/3 tracking)
- Nginx QUIC: https://nginx.org/en/docs/quic.html

---

## Updated from Research (2026-06-22)

### Go 1.26 `net/url` Breaking Change Affects URL-Validating Handlers

Gin itself does not call `url.Parse` on request URLs, but **handlers that validate user-supplied URLs** are affected by a Go 1.26 stdlib change:

- `url.Parse` and `url.ParseRequestURI` now **reject malformed URLs with multiple colons in the host subcomponent** (e.g. `http://localhost:8080:80/`, `http://::1/`). Previously these parsed silently with a malformed `Host` field. On Go 1.26 they return `*url.Error`.
- **Affected Gin handlers:** OAuth/OIDC redirect-URI validation, webhook URL intake, deep-link parsing, well-known endpoint discovery, link unfurlers. Anywhere your code does `url.Parse(someUserString)`.
- **Action for agents:** wrap `url.Parse` calls in handlers that take URLs from clients with explicit validation. The fix is also a security improvement (RFC 3986 compliance); the `GODEBUG=urlstrictcolon=0` opt-out exists for compatibility only.
  ```go
  // Wrong: passes malformed URLs through Go 1.26 will reject them
  u, err := url.Parse(userRedirectURI)
  
  // Right: explicit + better error for clients
  u, err := url.ParseRequestURI(userRedirectURI)
  if err != nil || u.Host == "" || u.Scheme != "https" {
      c.JSON(400, gin.H{"error": "invalid redirect_uri"})
      return
  }
  // Also: verify only one colon in Host (or none if no port)
  if strings.Count(u.Host, ":") > 1 && !strings.HasPrefix(u.Host, "[") {
      c.JSON(400, gin.H{"error": "invalid host"})
      return
  }
  ```
- Source: [go.dev/issue/75223](https://github.com/golang/go/issues/75223), [Go 1.26 release notes](https://go.dev/doc/go1.26).

### Gin v1.13 — `net/netip` IP Migration (PR #4599, in progress)

Gin v1.13 (milestone #28, due 2026-06-30, ~57% complete as of 2026-06-22) is migrating internal IP handling from `net.ParseIP` (`net.IP` / `[]byte`) to `net/netip` (`netip.Addr`):

- `c.ClientIP()` may return `netip.Addr` instead of `net.IP` in Gin v1.13.
- **Security note:** `netip.Prefix.Contains` does **not** auto-convert IPv4-mapped IPv6 addresses the way `net.IPNet.Contains` did. If you maintain a CIDR allowlist (rate-limit, geo-block, internal-network check) and switch from `net.ParseCIDR` to `netip.ParsePrefix`, **IPv4-mapped IPv6 addresses (`::ffff:1.2.3.4`) will no longer be contained by IPv4 CIDRs.** Either:
  1. Add explicit IPv6 CIDRs alongside IPv4 ones, OR
  2. Call `addr.Unmap()` on the address before the `Contains` check (converts IPv4-mapped IPv6 → IPv4), OR
  3. Validate that your service only ever sees one address family (typical for internet-facing APIs).
  ```go
  // MIGRATION GOTCHA: netip.Prefix.Contains differs from net.IPNet.Contains
  cidr := netip.MustParsePrefix("10.0.0.0/8")
  addr := netip.MustParseAddr("::ffff:10.1.2.3") // IPv4-mapped IPv6
  fmt.Println(cidr.Contains(addr))               // false (was true with net.IPNet!)
  fmt.Println(cidr.Contains(addr.Unmap()))        // true after explicit conversion
  ```
- **Action for agents:** when Gin v1.13 ships, audit any code that touches `c.ClientIP()` or maintains IP CIDR allowlists. Migrating to `netip` is generally a net win (smaller, immutable, no `nil`) but the IPv4-mapped IPv6 behavior change is a real security footgun if you don't account for it.

### Gin v1.13 — Trailing Slash Redirect Behavior Change (PR #4499, in progress)

Gin's current behavior 301-redirects `/foo/` → `/foo` (or vice-versa) depending on registration. PR #4499 changes this default. **Review your route registrations if you rely on the existing redirect behavior:**

- Current: `r.GET("/foo", ...)` and `r.GET("/foo/", ...)` are equivalent routes; Gin issues a 301 redirect from one to the other based on registration.
- After #4499: the redirect may be disabled or changed. Existing services that rely on the redirect (e.g. for SEO or canonical URL enforcement) may need to set `engine.RedirectTrailingSlash = true` explicitly.

### Gin v1.13 — Whole-Request Binding (PR #4543, in progress)

PR #4543 adds a "bind whole request at once" feature — bind JSON body, query params, headers, and URI params into a single struct in one `ShouldBind` call. Useful for thin CRUD handlers but currently no docs; track the PR for the merged API.

### Sources for This Update
- https://github.com/gin-gonic/gin/pull/4599 (net/netip migration)
- https://github.com/gin-gonic/gin/pull/4499 (trailing slash redirect)
- https://github.com/gin-gonic/gin/pull/4543 (whole-request binding)
- https://github.com/gin-gonic/gin/milestone/28 (Gin v1.13)
- https://github.com/golang/go/issues/75223 (url.Parse host colon rejection)
- https://go.dev/doc/go1.26 (Go 1.26 release notes)
- https://adam-p.ca/blog/2022/03/go-netip-flaw/ (IPv4-mapped IPv6 behavior in netip)

