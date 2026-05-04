# Handlers — HTTP Methods, JSON Binding, Validation

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

## Validation Tags (go-playground/validator)

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

## Common Mistakes

1. **Using `ShouldBindQuery` for POST body** — binds from URL query params, not body
2. **Not checking binding errors** — always validate `err != nil`
3. **Returning after `c.JSON()`** — Gin handlers don't auto-return, but you should return after sending response
4. **Wrong HTTP status codes** — 201 for create, 204 for delete with no body, 400 for bad request
5. **Not handling `c.Request.Body` EOF** — `ShouldBind` already handles this
6. **Multiple `ShouldBind` calls without `ShouldBindBodyWith`** — body is consumed on first bind