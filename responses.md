# Responses — JSON, XML, Streaming, ProtoBuf

## JSON Response

```go
// Map-based (quick)
c.JSON(200, gin.H{
    "message": "success",
    "data":    posts,
})

// Struct-based (recommended for typed responses)
type Response struct {
    Success bool        `json:"success"`
    Data    interface{} `json:"data,omitempty"`
    Error   string      `json:"error,omitempty"`
    Meta    *Meta       `json:"meta,omitempty"`
}

type Meta struct {
    Page  int `json:"page"`
    Total int `json:"total"`
}

c.JSON(200, Response{
    Success: true,
    Data:    posts,
    Meta:    &Meta{Page: 1, Total: 10},
})
```

## Error Responses

```go
// Standard error
c.JSON(400, gin.H{"error": "invalid request"})

// Validation errors (422)
c.JSON(422, gin.H{
    "error":   "validation failed",
    "details": map[string]string{
        "email": "must be a valid email",
        "age":   "must be at least 18",
    },
})

// Not found
c.JSON(404, gin.H{"error": "resource not found"})

// Unauthorized
c.JSON(401, gin.H{"error": "unauthorized"})

// Internal server error
c.JSON(500, gin.H{"error": "internal server error"})
```

## XML Response

```go
import "encoding/xml"

type Post struct {
    ID    int    `xml:"id"`
    Title string `xml:"title"`
    Body  string `xml:"body"`
}

c.XML(200, Post{ID: 1, Title: "Hello", Body: "World"})
```

## YAML Response

```go
import "gopkg.in/yaml.v3"

c.YAML(200, gin.H{"name": "gin", "version": "1.10"})
```

## ProtoBuf Response (Gin v1.12+)

```go
import "google.golang.org/protobuf/proto"

// Define protobuf message (in .proto file, compiled to .pb.go)
// message Response {
//   bool success = 1;
//   string message = 2;
//   Data data = 3;
// }

func protobufHandler(c *gin.Context) {
    // Negotiate ProtoBuf content type based on Accept header
    // Supports: application/json, application/xml, application/yaml, application/protobuf
    c.ProtoBuf(200, &Response{
        Success: true,
        Message: "hello",
    })
}

// Gin v1.12 content negotiation supports:
// - application/json (default)
// - application/xml
// - application/yaml
// - application/protobuf (NEW)
```

## HTML Response

```go
// From file
c.HTML(200, "index.html", gin.H{"title": "My Site"})

// From string
c.Data(200, "text/html; charset=utf-8", htmlContent)
```

## File Download

```go
// Download file
c.File("./exports/report.pdf")

// Custom filename
c.Header("Content-Disposition", "attachment; filename=report.pdf")
c.File("./exports/report.pdf")
```

## Streaming Responses

```go
// Server-Sent Events (SSE) — the idiomatic Gin way
// Use c.SSEvent() directly, not inside c.Stream()
func sseHandler(c *gin.Context) {
    c.Writer.Header().Set("Content-Type", "text/event-stream")
    c.Writer.Header().Set("Cache-Control", "no-cache")
    c.Writer.Header().Set("Connection", "keep-alive")

    for {
        select {
        case <-c.Request.Context().Done():
            return
        case msg := <-messages:
            c.SSEvent("message", msg)
            c.Writer.Flush()
        case <-time.After(30 * time.Second):
            // Keep-alive ping to prevent connection timeout
            c.SSEvent("ping", time.Now().Unix())
            c.Writer.Flush()
        }
    }
}

// Streaming JSON via c.Stream() — for NDJSON / JSON Lines
// c.Stream() sets Content-Type and handles flushing automatically
func streamPosts(c *gin.Context) {
    c.Stream(200, "application/json")
    for {
        posts, err := db.GetPostsBatched(100)
        if err != nil {
            return
        }
        if len(posts) == 0 {
            return
        }
        for _, post := range posts {
            c.SSEvent("post", post)
        }
        c.Writer.Flush()
        time.Sleep(100 * time.Millisecond)
    }
}

// Alternative: write NDJSON lines directly to the stream writer
func streamNDJSON(c *gin.Context) {
    c.Stream(200, "application/x-ndjson")
    for page := 1; ; page++ {
        posts, err := db.GetPostsBatchedPage(100, page)
        if err != nil || len(posts) == 0 {
            return
        }
        for _, post := range posts {
            b, _ := json.Marshal(post)
            c.Stream(func(w io.Writer) bool {
                w.Write(b)
                w.Write([]byte("\n"))
                return true // keep streaming
            })
        }
    }
}
```

**Key distinction:**
- `c.SSEvent()` — sends a formatted SSE message (`data: {...}\n\n`) to `c.Writer`; use it directly in loops
- `c.Stream()` — handles chunked transfer-encoding; pass it an `io.Writer` callback, not SSE event formatting

## Redirects

```go
// Temporary redirect (302)
c.Redirect(302, "/new-url")

// Permanent redirect (301)
c.Redirect(301, "/new-url")

// Route redirect
c.Redirect(302, r.GetRoute("posts").Path)
```

## Content Negotiation

```go
// Respond with format based on Accept header
// Gin v1.12+ supports: json, xml, yaml, protobuf
func respond(c *gin.Context) {
    content := gin.H{"message": "hello"}
    c.Negotiate(content) // uses Accept header to pick format
}

// Manual override
switch c.GetHeader("Accept") {
case "application/xml":
    c.XML(200, content)
case "application/yaml":
    c.YAML(200, content)
case "application/protobuf":
    c.ProtoBuf(200, content)
default:
    c.JSON(200, content)
}
```

## Pagination Response Pattern

```go
type PaginatedResponse struct {
    Success bool        `json:"success"`
    Data    interface{} `json:"data"`
    Meta    struct {
        Page       int `json:"page"`
        PerPage    int `json:"per_page"`
        Total      int `json:"total"`
        LastPage   int `json:"last_page"`
    } `json:"meta"`
}

func paginatedResponse(c *gin.Context, data interface{}, page, perPage, total int) {
    c.JSON(200, PaginatedResponse{
        Success: true,
        Data:    data,
        Meta: struct {
            Page     int `json:"page"`
            PerPage  int `json:"per_page"`
            Total    int `json:"total"`
            LastPage int `json:"last_page"`
        }{
            Page:     page,
            PerPage:  perPage,
            Total:    total,
            LastPage: (total + perPage - 1) / perPage,
        },
    })
}
```

## Common Mistakes

1. **Returning after `c.JSON()`** — while Gin technically handles this, always return for clarity
2. **Wrong content type for streaming** — set headers before streaming
3. **Not handling context cancellation in streams** — check `c.Request.Context().Done()`
4. **Using `gin.H{}` for everything** — struct types give you compile-time safety
5. **Not using ProtoBuf for high-performance gRPC integration** — Gin v1.12+ supports it natively via content negotiation
6. **Mixing `c.SSEvent()` inside `c.Stream()` callback** — `SSEvent` writes to `c.Writer`, not the Stream callback's `io.Writer` param; use `SSEvent` directly in loops instead
7. **No keep-alive ping in SSE** — long-lived SSE connections may be closed by proxies if no data flows for ~60s; send a periodic `ping` event
