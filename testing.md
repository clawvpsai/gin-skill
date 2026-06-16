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

// Using testify/mock with setup/teardown
func TestWithMockRepo(t *testing.T) {
    mockRepo := new(MockUserRepository)
    handler := NewUserHandler(mockRepo)
    
    t.Run("user found", func(t *testing.T) {
        mockRepo.On("FindByID", 1).Return(&User{ID: 1, Name: "Alice"}, nil)
        
        req, _ := http.NewRequest("GET", "/users/1", nil)
        w := httptest.NewRecorder()
        
        router := gin.New()
        router.GET("/users/:id", handler.GetUser)
        router.ServeHTTP(w, req)
        
        assert.Equal(t, http.StatusOK, w.Code)
        mockRepo.AssertExpectations(t)
    })
    
    t.Run("user not found", func(t *testing.T) {
        mockRepo.On("FindByID", 999).Return(nil, ErrNotFound)
        
        req, _ := http.NewRequest("GET", "/users/999", nil)
        w := httptest.NewRecorder()
        
        router := gin.New()
        router.GET("/users/:id", handler.GetUser)
        router.ServeHTTP(w, req)
        
        assert.Equal(t, http.StatusNotFound, w.Code)
        mockRepo.AssertExpectations(t)
    })
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
    
    // Cleanup — use transactions that rollback
    tx := db.Begin()
    tx.Exec("DELETE FROM users WHERE id = ?", user.ID)
    tx.Rollback() // ensures no leftover data even if test fails
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


## testing/synctest — Deterministic Concurrent Testing (Go 1.25+)

Go 1.25 introduced the stable `testing/synctest` package for deterministic testing of concurrent code. Previously required `GOEXPERIMENT=synctest` in Go 1.24.

```go
import "testing/synctest"
// Go 1.24: GOEXPERIMENT=synctest   |   Go 1.25+: stable, no build tag needed

// Before (non-deterministic, uses arbitrary sleeps):
func TestGoroutineCleanup(t *testing.T) {
    go func() { /* async work */ }()
    time.Sleep(100 * time.Millisecond)  // arbitrary wait — flaky!
}

// After (deterministic — waits for all goroutines to complete):
func TestGoroutineCleanup(t *testing.T) {
    synctest.Test(func() {
        go func() { /* async work */ }()
        // test waits here until all goroutines spawned in this function complete
    })
    // After synctest.Test returns, the goroutine world is clean
}
```

**Key functions:**
- `synctest.Test(f func())` — runs `f()` in an isolated goroutine world. Returns when all goroutines spawned within `f` have completed. Any goroutine that fails to terminate causes the test to fail.
- `synctest.Wait()` — called within `synctest.Test`, blocks until all goroutines spawned within that `synctest.Test` call have completed.

**Why use it:**
- Eliminates `time.Sleep()` in concurrent tests — no more arbitrary waits
- Catches goroutine leaks in tests before they become production bugs
- Complements the Go 1.26 goroutine leak profiler for production

**In Gin context — testing middleware with async operations:**
```go
func TestAsyncMiddleware(t *testing.T) {
    synctest.Test(func() {
        r := gin.New()
        r.Use(func(c *gin.Context) {
            go func() {
                time.Sleep(10 * time.Millisecond)
                c.Set("done", true)
            }()
            c.Next()
        })
        
        req, _ := http.NewRequest("GET", "/", nil)
        w := httptest.NewRecorder()
        r.ServeHTTP(w, req)
    })
    // All goroutines cleaned up by here
}
```

## Testify Assertions

```go
import "github.com/stretchr/testify/assert"

func TestAssertions(t *testing.T) {
    t.Helper() // mark this function as a helper; failures report caller's line
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
    
    // Collections (Go 1.21+ with slices)
    assert.EqualExportedValues(t, User{ID: 1, Name: "Alice"}, User{ID: 1, Name: "Alice"})
    assert.Subset(t, []int{1, 2, 3, 4}, []int{2, 3})
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

// Run benchmarks with: go test -bench=. -benchmem
// Go 1.24+: -benchtime=1x for quick smoke, -short for fast test runs
// Compare with: go test -bench=. -benchmem -cpuprofile=cpu.out
// Then: go tool pprof -http=:8080 cpu.out
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
10. **Not rolling back test transactions** — tests can leave stale data in the DB
11. **Using `assert.Equal` on slices with different order** — use `assert.ElementsMatch` or sort before comparing

---


## Modern `testing.TB` Methods (Go 1.24+ / 1.25+)

The `testing.TB` interface in Go 1.26 exposes several new methods that make Gin tests cleaner. These are part of the `T`, `B`, and `F` interfaces.

### `t.Context()` — Go 1.24+

Returns a `context.Context` canceled just before `Cleanup`-registered functions run. Use this instead of `context.Background()` in tests so resources tied to the test lifetime are released cleanly.

```go
func TestHandlerWithContext(t *testing.T) {
    r := gin.New()
    r.GET("/jobs/:id", func(c *gin.Context) {
        // Pass the test-managed context down to your service layer.
        // It is automatically canceled when the test ends — no leaks.
        job, err := jobService.Get(c.Request.Context(), c.Param("id"))
        if err != nil {
            c.JSON(500, gin.H{"error": err.Error()})
            return
        }
        c.JSON(200, job)
    })

    // For setup/teardown code that needs a context, prefer t.Context() over context.Background()
    ctx, cancel := context.WithCancel(t.Context())
    defer cancel()

    req, _ := http.NewRequestWithContext(ctx, "GET", "/jobs/42", nil)
    w := httptest.NewRecorder()
    r.ServeHTTP(w, req)
    assert.Equal(t, http.StatusOK, w.Code)
}
```

**In Gin handlers**, prefer `c.Request.Context()` (request-scoped, with client cancellation) over `t.Context()` (test-scoped). The two are different — `t.Context()` is for setup/teardown, not for handler logic.

### `t.Setenv()` — Go 1.17+ (always prefer this over `os.Setenv`)

Sets an env var and **automatically restores the original value** at test end. Cannot be used with `t.Parallel()` (env vars are process-wide).

```go
func TestConfigFromEnv(t *testing.T) {
    t.Setenv("DATABASE_URL", "postgres://test:test@localhost:5432/test")
    t.Setenv("JWT_SECRET", "test-secret")

    cfg, err := config.Load()
    assert.NoError(t, err)
    assert.Equal(t, "test-secret", cfg.JWTSecret)
    // env vars are auto-restored after the test — no manual cleanup
}
```

### `t.Chdir()` — Go 1.24+

Changes the working directory for the test and restores it on cleanup. Useful for tests that read config files relative to `.`.

```go
func TestLoadMigrations(t *testing.T) {
    t.Chdir("./testdata") // cwd is restored automatically

    // Now "./001_init.sql" resolves to "./testdata/001_init.sql"
    files, err := filepath.Glob("./*.sql")
    assert.NoError(t, err)
    assert.NotEmpty(t, files)
}
```

### `t.Attr()` / `t.ArtifactDir()` — Go 1.25+

`Attr` annotates the test with key/value metadata (visible in `go test -v` output and tooling). `ArtifactDir` returns a per-test directory for saving failure artifacts (screenshots, dumps, traces).

```go
func TestSlowEndpoint(t *testing.T) {
    t.Attr("endpoint", "/api/v1/heavy")
    t.Attr("expected_p95_ms", "200")

    // Save profiling data on failure for later inspection
    defer func() {
        if t.Failed() {
            dir := t.ArtifactDir()
            os.WriteFile(filepath.Join(dir, "trace.out"), profileData, 0644)
        }
    }()

    // ... test body ...
}
```

### `t.TempDir()` — Go 1.15+

Returns a unique temp directory cleaned up automatically. Use it for SQLite test DBs, file uploads, etc.

```go
func TestFileUpload(t *testing.T) {
    dir := t.TempDir() // auto-cleaned
    uploadPath := filepath.Join(dir, "upload.dat")

    // ... write file, test upload handler ...

    // No defer os.RemoveAll — TempDir handles it
}
```

## Fuzz Testing (Go 1.18+)

Gin handlers are ideal fuzz targets. Use `f.Fuzz` to feed random/seeded inputs.

```go
func FuzzCreatePost(f *testing.F) {
    // Seed corpus — must always be valid
    f.Add(`{"title": "Hello", "body": "World"}`)
    f.Add(`{}`)
    f.Add(`not json at all`)

    r := gin.New()
    r.POST("/posts", createPost)

    f.Fuzz(func(t *testing.T, body string) {
        req, _ := http.NewRequest("POST", "/posts", bytes.NewBufferString(body))
        req.Header.Set("Content-Type", "application/json")
        w := httptest.NewRecorder()

        // Must not panic, must return a valid HTTP status
        assert.NotPanics(t, func() { r.ServeHTTP(w, req) })
        assert.True(t, w.Code >= 200 && w.Code < 600)
    })
}

// Run with: go test -fuzz=FuzzCreatePost -fuzztime=30s
```

**What fuzzing catches in Gin handlers:**
1. Nil-pointer panics on unexpected input shapes
2. JSON unmarshal panics (recursion bombs, large numbers)
3. Path traversal in URL params
4. Integer overflow in size headers
5. Race conditions (Go 1.18+ with `-race` enabled)

## Test Shuffling (Go 1.20+)

Run tests in random order to catch dependencies on shared state. Critical for Gin apps where global middleware state can leak between tests.

```bash
# Run all tests in random order, seeded for reproducibility
go test -shuffle=1234567890 ./...

# Run with a fresh random seed each time
go test -shuffle=off ./...      # explicit order (default)
go test -shuffle=on ./...       # random each run
go test -shuffle=42 ./...       # seeded (reproducible)
```

**When to use `-shuffle=on`:**
- CI to catch order-dependent test failures
- After refactoring shared setup code
- When adding new tests to an existing suite

If a test fails under `-shuffle=on` but passes with `-shuffle=off`, you have order-dependence — almost always a sign of **shared state** (a global DB pool, a package-level `var`, or un-cleaned middleware state).

## Common Mistakes (Updated 2026-06)

12. **Using `os.Setenv` instead of `t.Setenv`** — leaks env state to other tests; restore is manual
13. **Using `context.Background()` in tests** — use `t.Context()` (Go 1.24+) so test cleanup can cancel outstanding work
14. **No fuzz tests on handler inputs** — fuzz catches panics and edge cases unit tests miss
15. **Running tests in fixed order** — use `go test -shuffle=on` in CI to expose state leaks
16. **Not using `t.TempDir()` for file-based tests** — leaves stale files across runs
17. **Not calling `t.Helper()` in assertion wrappers** — failure messages point to the wrapper, not the actual failing call
18. **Calling `t.Parallel()` after `t.Setenv`** — panics: env vars are process-wide

## Updated from Research (2026-05)

### Integration Tests Best Practices
- Use `testing.Short()` to skip integration tests with `go test -short`
- Use transactions that rollback — `tx.Begin()`, `tx.Rollback()` — so tests don't pollute the DB
- Keep integration tests in `_test.go` files with `_integration` suffix if needed
- Test database should be isolated per test — fresh DB per test suite or transactional rollback

### Table-Driven Tests with Subtests
- Subtests called by the same testing function run in series by default
- Use `t.Parallel()` inside `t.Run()` to parallelize subtests
- Subtests continue even if earlier subtest failed — catch all failures in one run

### Mocking Functions (not interfaces)
- For mocking individual functions, use interface wrappers or refactor to accept interfaces
- `go-sqlmock` for SQL testing, `go-redismock` for Redis
- Use `testify/mock` for cleaner setup/teardown with `.On()` / `.Return()` chains

### Benchmark Profiling
- Run benchmarks with `go test -bench=. -benchmem`
- Use `-cpuprofile` and `-memprofile` to identify bottlenecks
- `go tool pprof -http=:8080 cpu.out` provides an interactive web UI
- Use `-benchtime=1x` (Go 1.24+) for quick smoke benchmarks during dev

### Sources
- https://gin-gonic.com/en/docs/testing/
- https://pkg.go.dev/testing
- https://github.com/stretchr/testify