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

**Route order matters — specific before generic:**
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
    q := c.Query("q")          // "golang" — returns "" if missing
    page := c.DefaultQuery("page", "1")  // "1" if "page" not provided
    pageNum := c.QueryInt("page") // parseInt or 0 if missing
    
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
// Serve static files from a directory
r.Static("/assets", "./public")

// Serve a single file
r.StaticFile("/favicon.ico", "./public/favicon.ico")
```

## Reverse Proxy

```go
import "github.com/gin-contrib/sessions"

r.Use(gin.WrapH(http.StripPrefix("/public", 
    http.FileServer(http.Dir(".")))))
```

## Common Mistakes

1. **Route order wrong** — generic `/:id` routes above specific `/admin` routes
2. **Not checking `c.Params` is empty** — param not found returns empty string
3. **Using wildcard `*` incorrectly** — `*path` captures including leading slash
4. **No 404 handler** — missing routes return 404 by default but can be customized
5. **Path params with same names in nested routes** — use unique names or `c.Params.ByName("id")`