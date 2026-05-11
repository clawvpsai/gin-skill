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
    "github.com/gin-gonic/gin"
    "github.com/quic-go/quic-go/http3"
)

// Option 1: Run with HTTP/3 support (Gin v1.11+)
// Requires quic-go dependency: go get github.com/quic-go/quic-go
func runHTTP3(r *gin.Engine, addr string) error {
    server := &http3.Server{
        Addr:    addr,
        Handler: r,
    }
    return server.ListenAndServe()
}

// Option 2: Dual HTTP/1.1 + HTTP/3 server
// Start HTTP/1.1 in a goroutine, HTTP/3 on the same port (if supported)
// In most setups, put Nginx with HTTP/3 in front of Gin

// Note: HTTP/3 benefits are most visible on unreliable networks (mobile, high-latency).
// For most internal/microservice setups, HTTP/1.1 or HTTP/2 behind a load balancer is sufficient.
```

**When to use HTTP/3:**
- Public APIs with diverse client networks (mobile, international)
- Real-time applications (WebSocket alternatives)
- When behind a HTTP/3-capable reverse proxy (Nginx 1.25+, Caddy)

**When to skip:**
- Internal services behind a load balancer that already handles HTTP/2
- Latency-insensitive local networks
- When `quic-go` dependency complexity isn't worth the gain

## Common Mistakes

1. **Route order wrong** — generic `/:id` routes above specific `/admin` routes
2. **Not checking `c.Params` is empty** — param not found returns empty string
3. **Using wildcard `*` incorrectly** — `*path` captures including leading slash
4. **No 404 handler** — missing routes return 404 by default but can be customized
5. **Path params with same names in nested routes** — use unique names or `c.Params.ByName("id")`
6. **Not using escaped path routing when URL encoding matters** — set `r.UseEscapedPath = true` on the engine in Gin v1.12+
7. **Using `c.QueryInt()`** — doesn't exist in Gin, use `strconv.Atoi(c.Query("page"))` instead
8. **Static file setup without `net/http` import** — `http.ServeFile` needs the import
