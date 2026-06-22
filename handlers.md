# Handlers — HTTP Methods, JSON Binding, Validation

> **Performance tip (Go 1.27+):** Gin's `c.JSON()` / `c.ShouldBindJSON()` use `encoding/json` v1. For 2.7–10.2× faster unmarshaling, register a custom `render.Render` for json/v2 (see `responses.md` → "JSON v2 Custom Renderer"). For request binding with v2, replace `c.ShouldBindJSON` with `jsonv2.UnmarshalRead(c.Request.Body, &req)`.

> **Go 1.26 breaking change:** `url.Parse` and `url.ParseRequestURI` now reject malformed URLs with extra colons in the host subcomponent (e.g. `http://localhost:8080:80/`). Affects handlers that validate user-supplied URLs (OAuth/OIDC redirect_uri, webhook intake, link unfurlers). Use `url.ParseRequestURI` and validate `u.Host` explicitly. See `routing.md` → "Updated from Research (2026-06-22)" for the full pattern.

## Handler Structure

```go
func getPosts(c *gin.Context) {
    posts, err := db.GetPosts()
    if err != nil {
        c.JSON(500, gin.H{"error": "failed to fetch posts"})
        return
    }
    c.JSON(200, posts)
}
```

## JSON Binding

```go
// POST /posts with JSON body
type CreatePostRequest struct {
    Title   string `json:"title" binding:"required,min=3,max=100"`
    Body    string `json:"body" binding:"required,min=10"`
    Tags    []int  `json:"tags"`
    Publish bool   `json:"publish"`
}

func createPost(c *gin.Context) {
    var req CreatePostRequest
    
    // Binds and validates JSON in one step
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(400, gin.H{"error": err.Error()})
        return
    }
    
    post, err := db.CreatePost(req.Title, req.Body, req.Tags)
    if err != nil {
        c.JSON(500, gin.H{"error": "failed to create post"})
        return
    }
    
    // 201 Created
    c.JSON(201, post)
}
```

## Form Binding

```go
// POST form data (Content-Type: application/x-www-form-urlencoded)
type LoginRequest struct {
    Email    string `form:"email" binding:"required,email"`
    Password string `form:"password" binding:"required,min=8"`
}

func login(c *gin.Context) {
    var req LoginRequest
    if err := c.ShouldBind(&req); err != nil {
        c.JSON(400, gin.H{"error": err.Error()})
        return
    }
    // ...
}
```

## Query Binding vs Body Binding

| Method | Source | Use When |
|--------|--------|----------|
| `ShouldBindQuery` | URL query params (`?foo=bar`) | GET requests, filters |
| `ShouldBind` | Request body (JSON/form) | POST/PUT with body |
| `ShouldBindJSON` | JSON body only | Explicit JSON POST |
| `ShouldBindUri` | Path params (`:id`) | Route parameters |
| `ShouldBindHeader` | Request headers | Header extraction |

```go
// Query binding — reads from ?email=...&password=...
type SearchRequest struct {
    Query  string `form:"q"`
    Page   int    `form:"page,default=1"`
    Limit  int    `form:"limit,default=20"`
}

func search(c *gin.Context) {
    var req SearchRequest
    if err := c.ShouldBindQuery(&req); err != nil {
        c.JSON(400, gin.H{"error": err.Error()})
        return
    }
    // req.Query, req.Page, req.Limit are populated
}

// WRONG: ShouldBindQuery for POST body
func wrong(c *gin.Context) {
    var req struct { Email string `form:"email"` }
    c.ShouldBindQuery(&req) // Binds from URL query, NOT body — likely wrong for POST
}

// RIGHT: ShouldBind for POST body
func right(c *gin.Context) {
    var req struct { Email string `json:"email"` }
    c.ShouldBindJSON(&req) // Binds from JSON body
}
```

## URI Parameters (path binding)

```go
type PostIDParam struct {
    ID int `uri:"id" binding:"required"`
}

func getPost(c *gin.Context) {
    var param PostIDParam
    if err := c.ShouldBindUri(&param); err != nil {
        c.JSON(400, gin.H{"error": "invalid id"})
        return
    }
    
    post, err := db.GetPost(param.ID)
    if err != nil {
        c.JSON(404, gin.H{"error": "post not found"})
        return
    }
    c.JSON(200, post)
}
```

**⚠️ Gotcha — `binding:"required"` on numeric types:** For `int`/`uint` fields, `required` only fails on the zero value. If `0` is a valid ID in your domain, this won't catch missing values. Use a range constraint instead:

```go
// Bad — only fails if ID == 0
type BadParam struct {
    ID int `uri:"id" binding:"required"`
}

// Good — explicit range; also rejects negative IDs
type GoodParam struct {
    ID int `uri:"id" binding:"required,gt=0"`
}
```

This applies to **all numeric types**: `int`, `int64`, `float64`, `uint`, etc. For string IDs, `required` works as expected (rejects empty string).

## Multiple Body Reads — ShouldBindBodyWith

Gin's `ShouldBind*` methods consume `c.Request.Body`. If you need to bind the same body multiple times (e.g., logging + processing), use `ShouldBindBodyWith`:

```go
func createPost(c *gin.Context) {
    var req CreatePostRequest
    
    // First bind attempt
    if err := c.ShouldBindBodyWith(&req, binding.JSON); err != nil {
        c.JSON(400, gin.H{"error": err.Error()})
        return
    }
    
    // Body is still available for subsequent reads
    // (underlying buffer is restored)
    
    // You can also bind to a different type
    var raw map[string]interface{}
    c.ShouldBindBodyWith(&raw, binding.JSON)
    
    c.JSON(201, gin.H{"created": true})
}
```

**Supported content types for `ShouldBindBodyWith`:**
- `binding.JSON` — `application/json`
- `binding.XML` — `application/xml`
- `binding.Form` — `application/x-www-form-urlencoded`
- `binding.FormMultipart` — `multipart/form-data`
- `binding.YAML` — `application/yaml`
- `binding.ProtoBuf` — `application/protobuf`

## Validation Tags (use `binding` tag — backed by go-playground/validator)

```go
type CreateUserRequest struct {
    Name     string `json:"name" binding:"required,min=2,max=50"`
    Email    string `json:"email" binding:"required,email"`
    Age      int    `json:"age" binding:"omitempty,min=18,max=120"`
    Password string `json:"password" binding:"required,min=8"`
    Website  string `json:"website" binding:"omitempty,url"`
    Phone    string `json:"phone" binding:"omitempty,e164"` // E.164 format
}

// Common tags
// required, omitempty, email, url, min, max, len
// eq, ne, gt, gte, lt, lte
// uuid, alphanum, numeric
// unique (for slices)
```

## Custom Validation

```go
// Register custom validator
func customValidation() {
    if v, ok := binding.Validator.Engine().(*validator.Validate); ok {
        v.RegisterValidation("valid-status", func(fl validator.FieldLevel) bool {
            return fl.Field().String() == "active" || fl.Field().String() == "inactive"
        })
    }
}

type UpdateStatusRequest struct {
    Status string `json:"status" binding:"required,valid-status"`
}
```

## Reading Request Headers

```go
func handler(c *gin.Context) {
    auth := c.GetHeader("Authorization")
    contentType := c.GetHeader("Content-Type")
    requestID := c.GetHeader("X-Request-ID")
    accept := c.GetHeader("Accept")
}
```

## Reading Request Body

```go
// Read body as bytes
body, _ := io.ReadAll(c.Request.Body)
// Re-use body by putting it back
c.Request.Body = io.NopCloser(bytes.NewBuffer(body))

// Or use ShouldBindBodyWith for multiple reads
c.ShouldBindBodyWith(&req, binding.JSON)
```

## WebSocket Upgrades

Gin does not ship with WebSocket support, but the upgrade pattern is short. Two library options:

- **`github.com/gorilla/websocket`** — the de-facto standard, used in most production Go codebases. Years of production hardening, broad Stack Overflow coverage, works with Go 1.20+.
- **`github.com/coder/websocket`** (formerly `nhooyr.io/websocket`) — modern, minimal API, context-first. Cleaner for new code if you can adopt it.

### Basic Upgrade (gorilla/websocket)

```go
import (
    "net/http"
    "github.com/gin-gonic/gin"
    "github.com/gorilla/websocket"
)

// Upgrader is safe to reuse across requests — it is stateless
var upgrader = websocket.Upgrader{
    ReadBufferSize:  1024,
    WriteBufferSize: 1024,
    // CheckOrigin prevents CSWSH (Cross-Site WebSocket Hijacking).
    // In production, validate the Origin header against an allowlist.
    CheckOrigin: func(r *http.Request) bool {
        origin := r.Header.Get("Origin")
        return origin == "" || origin == "https://myapp.com" || origin == "https://admin.myapp.com"
    },
}

func wsHandler(c *gin.Context) {
    conn, err := upgrader.Upgrade(c.Writer, c.Request, nil)
    if err != nil {
        // Upgrade already wrote an error response; just return.
        return
    }
    defer conn.Close()

    for {
        msgType, msg, err := conn.ReadMessage()
        if err != nil {
            return // client disconnected
        }
        if err := conn.WriteMessage(msgType, msg); err != nil {
            return
        }
    }
}

// Wire it up — Gin treats it like any other route
r.GET("/ws", wsHandler)
```

### Production Pattern — Read/Write Deadlines + Ping/Pong

Without deadlines, a dead network connection leaks forever. Without ping/pong, intermediaries drop idle connections in ~60 seconds. Always use both:

```go
const (
    writeWait      = 10 * time.Second
    pongWait       = 60 * time.Second
    pingPeriod     = (pongWait * 9) / 10 // send pings at 54s — must be < pongWait
    maxMessageSize = 8192
)

func wsHandler(c *gin.Context) {
    conn, err := upgrader.Upgrade(c.Writer, c.Request, nil)
    if err != nil {
        return
    }
    defer conn.Close()

    conn.SetReadLimit(maxMessageSize)
    _ = conn.SetReadDeadline(time.Now().Add(pongWait))
    conn.SetPongHandler(func(string) error {
        _ = conn.SetReadDeadline(time.Now().Add(pongWait))
        return nil
    })

    // Reader goroutine: detect client disconnects via read errors
    done := make(chan struct{})
    go func() {
        defer close(done)
        for {
            _, msg, err := conn.ReadMessage()
            if err != nil {
                return
            }
            // Handle message here (or push to a channel for the writer)
            _ = msg
        }
    }()

    // Writer goroutine: periodic ping keeps the connection alive
    ticker := time.NewTicker(pingPeriod)
    defer ticker.Stop()
    for {
        select {
        case <-done:
            return
        case <-ticker.C:
            _ = conn.SetWriteDeadline(time.Now().Add(writeWait))
            if err := conn.WriteMessage(websocket.PingMessage, nil); err != nil {
                return
            }
        }
    }
}
```

### Hub Pattern — Broadcasting to Multiple Clients

For chat rooms, notifications, live dashboards — fan-out from a single source to many connections:

```go
type Hub struct {
    clients    map[*websocket.Conn]bool
    broadcast  chan []byte
    register   chan *websocket.Conn
    unregister chan *websocket.Conn
    mu         sync.RWMutex
}

func NewHub() *Hub {
    return &Hub{
        broadcast:  make(chan []byte, 256),
        register:   make(chan *websocket.Conn),
        unregister: make(chan *websocket.Conn),
        clients:    make(map[*websocket.Conn]bool),
    }
}

func (h *Hub) Run() {
    for {
        select {
        case conn := <-h.register:
            h.mu.Lock()
            h.clients[conn] = true
            h.mu.Unlock()
        case conn := <-h.unregister:
            h.mu.Lock()
            if _, ok := h.clients[conn]; ok {
                delete(h.clients, conn)
                conn.Close()
            }
            h.mu.Unlock()
        case msg := <-h.broadcast:
            h.mu.RLock()
            for conn := range h.clients {
                _ = conn.SetWriteDeadline(time.Now().Add(writeWait))
                if err := conn.WriteMessage(websocket.TextMessage, msg); err != nil {
                    // write failed — schedule cleanup via unregister
                    go func(c *websocket.Conn) { h.unregister <- c }(conn)
                }
            }
            h.mu.RUnlock()
        }
    }
}

// Handler: each connection registers with the hub and pumps its own messages
func (h *Hub) ServeWS(c *gin.Context) {
    conn, err := upgrader.Upgrade(c.Writer, c.Request, nil)
    if err != nil {
        return
    }
    h.register <- conn
    // Reader: forward client messages into the broadcast channel
    go func() {
        defer func() { h.unregister <- conn }()
        conn.SetReadLimit(maxMessageSize)
        for {
            _, msg, err := conn.ReadMessage()
            if err != nil {
                return
            }
            h.broadcast <- msg
        }
    }()
}

// Wire up: hub.Run() once at startup
// r.GET("/ws", hub.ServeWS)
```

### coder/websocket (modern alternative)

```go
import "github.com/coder/websocket"

func wsHandler(w http.ResponseWriter, r *http.Request) {
    // Accept takes a context — cancellation propagates to the conn
    conn, err := websocket.Accept(w, r, &websocket.AcceptOptions{
        OriginPatterns: []string{"myapp.com", "*.myapp.com"},
    })
    if err != nil {
        return
    }
    defer conn.Close(websocket.StatusNormalClosure, "bye")

    ctx := r.Context()
    for {
        msgType, data, err := conn.Read(ctx)
        if err != nil {
            return
        }
        if err := conn.Write(ctx, msgType, data); err != nil {
            return
        }
    }
}
```

**Key differences from gorilla/websocket:**
- `Accept` takes the request context — when the client disconnects, `Read` returns immediately. No need for a separate ping/pong goroutine.
- `OriginPatterns` uses wildcard subdomain matching — cleaner than a `CheckOrigin` callback.
- Built-in `Close(code, reason)` returns a clean WebSocket close frame with a status code.

### WebSocket Behind Gin — Notes

- **CORS + WebSockets**: `CheckOrigin` (gorilla) or `OriginPatterns` (coder) handles the Origin check. You do NOT also need a `cors` middleware on the `/ws` route — WebSocket handshakes bypass CORS but the browser still sends `Origin`.
- **Gin middleware runs before upgrade**: auth, request ID, logging middleware all run as usual on the GET request. The upgrade happens inside the handler. If auth fails, return 401 before calling `upgrader.Upgrade`.
- **Hijacking**: `upgrader.Upgrade` calls `http.Hijacker` on the underlying `http.ResponseWriter` — Gin's writer supports this, but be aware that after upgrade, Gin no longer mediates the connection.
- **Sticky sessions / load balancers**: WebSocket connections are long-lived. Behind a load balancer, configure sticky sessions (cookie-based) or run the WS endpoints on a dedicated hostname that bypasses the LB.
- **Graceful shutdown**: `conn.Close()` from the shutdown signal will not flush in-flight messages. Track connections in a `sync.WaitGroup` and call `conn.WriteControl(websocket.CloseMessage, websocket.FormatCloseMessage(websocket.CloseGoingAway, ""), time.Now().Add(time.Second))` to send a clean close frame.

## Gin v1.13 In-Flight Changes Affecting Handlers (Verify Before Upgrade)

Gin v1.13 (milestone #28, due 2026-06-30, ~57% complete as of 2026-06-22) has several open PRs that change handler behavior. Watch these when v1.13 ships:

- **`fix(form): correctly differentiate between nil / present-but-empty slices`** (PR #4498) — form binding currently cannot distinguish `?tags=` (present but empty string) from `?tags` (missing entirely); both produce a `nil` slice. After the fix, `tags=[]` vs `tags=nil` will be distinguishable. **Action:** audit form-binding handlers that rely on the current `nil == empty` semantics. Existing tests that compare against `nil` will need updating.
- **`feat(binding): add support for binding whole request at once`** (PR #4543) — one `ShouldBind` call will populate a single struct from JSON body, query params, headers, and URI params simultaneously. Useful for thin CRUD handlers; the exact API is not yet documented. Track for merged API before adopting.
- **`Changed trailing slash redirection behaviour`** (PR #4499) — Gin's 301 redirect between `/foo` and `/foo/` may change defaults. If your service relies on the current redirect for SEO/canonical URLs, set `engine.RedirectTrailingSlash = true` explicitly post-upgrade.
- **`chore(response_writer): add Unwrap() method to ResponseWriter interface`** (PR #4506) — enables `errors.As(err, &http.ResponseWriter)` and `errors.Is(err, http.ErrHandlerTimeout)` to traverse the Gin writer chain. Low impact but improves error inspection for handler-timeout logic.

**For now (Gin v1.12.x):** none of these changes are applied yet. v1.12.x behavior is preserved.

See `routing.md` → "Updated from Research (2026-06-22)" for the `net/netip` IP migration (PR #4599) which also affects `c.ClientIP()` consumers.


## Common Mistakes

1. **Using `ShouldBindQuery` for POST body** — binds from URL query params, not body
2. **Not checking binding errors** — always validate `err != nil`
3. **Returning after `c.JSON()`** — Gin handlers don't auto-return, but you should return after sending response
4. **Wrong HTTP status codes** — 201 for create, 204 for delete with no body, 400 for bad request
5. **Not handling `c.Request.Body` EOF** — `ShouldBind` already handles this
6. **Multiple `ShouldBind` calls without `ShouldBindBodyWith`** — body is consumed on first bind
7. **`binding:"required"` on numeric types** — silently accepts the zero value. Use `binding:"required,gt=0"` (or `gte=1`) for IDs/positive integers. Strings are unaffected — `required` correctly rejects empty strings.
8. **WebSocket: `CheckOrigin` returning `true` always** — disables CSWSH protection. Always validate `Origin` against an allowlist, or use `coder/websocket`'s `OriginPatterns`.
9. **WebSocket: no read deadline** — a dead network connection leaks the goroutine forever. Always call `conn.SetReadDeadline(...)` and refresh it in the pong handler.
10. **WebSocket: no write deadline or ping interval** — intermediaries drop idle connections in ~60s. Send a `PingMessage` every ~30s with a write deadline.
11. **WebSocket: `conn.Close()` on shutdown** — does not send a clean close frame. Use `conn.WriteControl(websocket.CloseMessage, websocket.FormatCloseMessage(websocket.CloseGoingAway, ""), deadline)` first, then `Close`.
12. **WebSocket: reading and writing from the same goroutine** — gorilla/websocket allows only one concurrent reader and one concurrent writer. Serialize writes with a single writer goroutine, or guard with a mutex.