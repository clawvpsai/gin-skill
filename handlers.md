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

1. **Using `c.Query()` for POST body** — use `ShouldBind` for JSON/form data
2. **Not checking binding errors** — always validate `err != nil`
3. **Returning after `c.JSON()`** — Gin handlers don't auto-return, but you should return after sending response
4. **Wrong HTTP status codes** — 201 for create, 204 for delete with no body, 400 for bad request
5. **Not handling `c.Request.Body` EOF** — `ShouldBind` already handles this