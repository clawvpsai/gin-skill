# Testing — Unit Tests, httptest, Mocking

## Test Setup

```go
package handlers_test

import (
    "bytes"
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


## Updated from Research (2026-05)

- **Testing | Gin Web Framework** (https://gin-gonic.com/en/docs/testing/)
  The net/http/httptest package is the preferred way for HTTP testing. Call gin.SetMode(gin.TestMode) before creating the router in your tests. This suppresses the debug-level route registration logs that Gin prints by default, keeping your test output clean. You can place this in TestMain so it applies to all tests in the package: ... Table-driven tests let you cover many input/output combinations without duplicating test logic. This pattern

- **Testing in Go Best Practices and Tips** (https://grid.gg/testing-in-go-best-practices-and-tips/)
  Subtests called by the same testing function will, generally, run in series, and will run through all subtests even if an earlier subtest has failed. In the cases where one would prefer the tests to run in parallel there’s instead the ... t.Run is a good practice as it allows the testing framework to better utilize its available resources.

- **r/golang on Reddit: Best practice testing** (https://www.reddit.com/r/golang/comments/1dvecs4/best_practice_testing/)
  21 votes, 56 comments. I&#x27;ve built an application in go. I&#x27;m now writing tests for it. To mock functions, it seems I need to refactor to use func
