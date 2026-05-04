# Testing — Unit Tests, Integration Tests, httptest, Mocking

## Test Setup

```go
package handlers_test

import (
    "bytes"
    "context"
    "encoding/json"
    "net/http"
    "net/http/httptest"
    "testing"
    
    "github.com/gin-gonic/gin"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/mock"
)

// TestMain — setup/teardown for all tests
func TestMain(m *testing.M) {
    gin.SetMode(gin.TestMode)
    m.Run()
}
```

## Testing Handlers

```go
func TestGetPosts(t *testing.T) {
    // Setup router
    r := gin.New()
    r.GET("/posts", getPosts)
    
    // Create request
    req, _ := http.NewRequest("GET", "/posts", nil)
    req.Header.Set("Accept", "application/json")
    
    // Record response
    w := httptest.NewRecorder()
    r.ServeHTTP(w, req)
    
    // Assertions
    assert.Equal(t, http.StatusOK, w.Code)
    
    var response map[string]interface{}
    err := json.Unmarshal(w.Body.Bytes(), &response)
    assert.NoError(t, err)
    assert.Contains(t, response, "data")
}

func TestCreatePost(t *testing.T) {
    r := gin.New()
    r.POST("/posts", createPost)
    
    body := `{"title": "Hello", "body": "World"}`
    req, _ := http.NewRequest("POST", "/posts", bytes.NewBufferString(body))
    req.Header.Set("Content-Type", "application/json")
    
    w := httptest.NewRecorder()
    r.ServeHTTP(w, req)
    
    assert.Equal(t, http.StatusCreated, w.Code)
}

func TestCreatePostValidation(t *testing.T) {
    r := gin.New()
    r.POST("/posts", createPost)
    
    // Missing required fields
    body := `{"title": "Hi"}`
    req, _ := http.NewRequest("POST", "/posts", bytes.NewBufferString(body))
    req.Header.Set("Content-Type", "application/json")
    
    w := httptest.NewRecorder()
    r.ServeHTTP(w, req)
    
    assert.Equal(t, http.StatusBadRequest, w.Code)
}
```

## Table-Driven Tests

```go
func TestValidation(t *testing.T) {
    tests := []struct {
        name    string
        input   string
        wantErr bool
    }{
        {"valid email", `{"email": "test@example.com"}`, false},
        {"invalid email", `{"email": "notanemail"}`, true},
        {"missing email", `{}`, true},
        {"empty body", ``, true},
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            r := gin.New()
            r.POST("/validate", validateHandler)
            
            req, _ := http.NewRequest("POST", "/validate", bytes.NewBufferString(tt.input))
            req.Header.Set("Content-Type", "application/json")
            
            w := httptest.NewRecorder()
            r.ServeHTTP(w, req)
            
            if tt.wantErr {
                assert.True(t, w.Code >= 400)
            } else {
                assert.Equal(t, http.StatusOK, w.Code)
            }
        })
    }
}
```

## Testing Middleware

```go
func TestJWTAuthMiddleware(t *testing.T) {
    r := gin.New()
    r.Use(JWTAuthMiddleware())
    r.GET("/protected", func(c *gin.Context) {
        userID, _ := c.Get("user_id")
        c.JSON(200, gin.H{"user_id": userID})
    })
    
    t.Run("no token", func(t *testing.T) {
        req, _ := http.NewRequest("GET", "/protected", nil)
        w := httptest.NewRecorder()
        r.ServeHTTP(w, req)
        
        assert.Equal(t, http.StatusUnauthorized, w.Code)
    })
    
    t.Run("invalid token", func(t *testing.T) {
        req, _ := http.NewRequest("GET", "/protected", nil)
        req.Header.Set("Authorization", "Bearer invalid")
        w := httptest.NewRecorder()
        r.ServeHTTP(w, req)
        
        assert.Equal(t, http.StatusUnauthorized, w.Code)
    })
    
    t.Run("valid token", func(t *testing.T) {
        token, _ := GenerateToken(123, "test@example.com")
        
        req, _ := http.NewRequest("GET", "/protected", nil)
        req.Header.Set("Authorization", "Bearer "+token)
        w := httptest.NewRecorder()
        r.ServeHTTP(w, req)
        
        assert.Equal(t, http.StatusOK, w.Code)
        
        var response map[string]interface{}
        json.Unmarshal(w.Body.Bytes(), &response)
        assert.Equal(t, float64(123), response["user_id"])
    })
}
```

## Mocking

```go
import "github.com/stretchr/testify/mock"

type MockDB struct {
    mock.Mock
}

func (m *MockDB) GetUser(id int) (*User, error) {
    args := m.Called(id)
    if args.Get(0) == nil {
        return nil, args.Error(1)
    }
    return args.Get(0).(*User), args.Error(1)
}

func TestGetUser(t *testing.T) {
    mockDB := new(MockDB)
    
    expectedUser := &User{ID: 1, Name: "John"}
    mockDB.On("GetUser", 1).Return(expectedUser, nil)
    
    // In handler, inject mockDB via dependency injection
    user, err := mockDB.GetUser(1)
    
    assert.NoError(t, err)
    assert.Equal(t, "John", user.Name)
    mockDB.AssertExpectations(t)
}
```

## Integration Tests (Test Database)

```go
func TestIntegration_CreateAndFetchUser(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test in short mode")
    }
    
    // Use a separate test database
    dsn := os.Getenv("TEST_DATABASE_URL")
    if dsn == "" {
        t.Skip("TEST_DATABASE_URL not set")
    }
    
    db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{})
    if err != nil {
        t.Fatalf("failed to connect test db: %v", err)
    }
    
    // Run migrations
    migrateDB(db)
    
    // Create user via service
    user, err := userService.Create(context.Background(), &CreateUserRequest{
        Email:    "test@example.com",
        Password: "password123",
    })
    assert.NoError(t, err)
    
    // Fetch and verify
    fetched, err := userService.GetByID(context.Background(), user.ID)
    assert.NoError(t, err)
    assert.Equal(t, "test@example.com", fetched.Email)
    
    // Cleanup
    db.Exec("DELETE FROM users WHERE id = ?", user.ID)
}
```

## Subtests with Parallel Execution

```go
func TestUserHandlers(t *testing.T) {
    r := gin.New()
    r.POST("/users", createUser)
    r.GET("/users/:id", getUser)
    
    tests := []struct {
        name           string
        method         string
        path           string
        body           string
        wantStatus     int
        setupToken     bool
    }{
        {
            name:       "create valid user",
            method:     "POST",
            path:       "/users",
            body:       `{"email":"test@example.com","password":"password123"}`,
            wantStatus: http.StatusCreated,
        },
        {
            name:       "create duplicate email",
            method:     "POST",
            path:       "/users",
            body:       `{"email":"test@example.com","password":"password123"}`,
            wantStatus: http.StatusConflict,
        },
        {
            name:       "get nonexistent user",
            method:     "GET",
            path:       "/users/999999",
            wantStatus: http.StatusNotFound,
        },
    }
    
    for _, tt := range tests {
        tt := tt // capture range variable
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel() // run subtests in parallel
            
            var body *bytes.Buffer
            if tt.body != "" {
                body = bytes.NewBufferString(tt.body)
            } else {
                body = bytes.NewBuffer(nil)
            }
            
            req, _ := http.NewRequest(tt.method, tt.path, body)
            req.Header.Set("Content-Type", "application/json")
            
            w := httptest.NewRecorder()
            r.ServeHTTP(w, req)
            
            assert.Equal(t, tt.wantStatus, w.Code)
        })
    }
}
```

## Testify Assertions

```go
import "github.com/stretchr/testify/assert"

func TestAssertions(t *testing.T) {
    // Equality
    assert.Equal(t, "expected", "actual")
    assert.NotEqual(t, 1, 2)
    
    // Nil/empty
    assert.Nil(t, nil)
    assert.NotNil(t, obj)
    assert.Empty(t, "")
    assert.NotEmpty(t, "value")
    
    // Booleans
    assert.True(t, true)
    assert.False(t, false)
    
    // Contains
    assert.Contains(t, "hello world", "world")
    
    // Type
    assert.IsType(t, (*User)(nil), user)
    
    // Errors
    assert.NoError(t, err)
    assert.Error(t, err)
    assert.EqualError(t, err, "expected error message")
}
```

## Benchmarking

```go
func BenchmarkGetPosts(b *testing.B) {
    r := gin.New()
    r.GET("/posts", getPosts)
    
    req, _ := http.NewRequest("GET", "/posts", nil)
    
    for i := 0; i < b.N; i++ {
        w := httptest.NewRecorder()
        r.ServeHTTP(w, req)
    }
}
```

## Common Mistakes

1. **Not setting `gin.SetMode(gin.TestMode)`** — tests run in debug mode, may behave differently
2. **Testing with real DB** — use mocks or test database
3. **Not checking response body** — status 200 doesn't mean correct data
4. **Using real HTTP client** — use `httptest.NewRecorder()` instead
5. **Mocking the wrong interface** — match actual interface signature
6. **Not asserting on all important fields** — check status, body, headers
7. **Not running subtests in parallel** — `t.Run` with `t.Parallel()` for faster test suites
8. **No integration test skip** — use `testing.Short()` to skip slow integration tests
9. **Sharing state between tests** — each test should be independent, no shared `var db *gorm.DB`

---

## Updated from Research (2026-05)

### Integration Tests Best Practices
- Use `testing.Short()` to skip integration tests with `go test -short`
- Keep integration tests in `_test.go` files with `_integration` suffix if needed
- Test database should be isolated per test — use transactions that rollback or fresh DB per test suite

### Table-Driven Tests with Subtests
- Subtests called by the same testing function run in series by default
- Use `t.Parallel()` inside `t.Run()` to parallelize subtests
- Subtests continue even if earlier subtest fails — catch all failures in one run

### Mocking Functions (not interfaces)
- For mocking individual functions, use interface wrappers or refactor to accept interfaces
- `go-sqlmock` for SQL testing, `go-redismock` for Redis

### Sources
- https://gin-gonic.com/en/docs/testing/
- https://grid.gg/testing-in-go-best-practices-and-tips/