# Go/Gin Version-Specific Requirements

## Active Go Versions

- **Go 1.24** — Current stable (released February 2025)
- **Go 1.23** — Previous stable
- **Go 1.22** — Still supported

## Version Selector Prompt

When working with Gin, check Go version first:
```bash
go version | grep -oP 'go\d+\.\d+'
```

Then load the relevant version sections below.

---

## Go 1.24 (Current — February 2025)

### New in Go 1.24

- **Generic type aliases** — type aliases may now be parameterized
  ```go
  type List[T any] = []T  // now valid!
  ```
- **Improved `go test -cpuprofile`** profiling
- **`math/rand/v2`** — new v2 API for random numbers
- **Toolchain improvements** — better `go mod` handling
- **Performance optimizations** — refined garbage collection

### Gin-Specific (Gin v1.10+)

- Use `gin.Context.Param()` — works with all Go versions
- `gin.H{}` is just `map[string]interface{}` — no changes needed

---

## Go 1.23 (2024)

### New in Go 1.23

- **Range-over-func** — iterate over functions/generators
  ```go
  for v := range func(yield func(int) bool) {
      yield(1)
      yield(2)
      yield(3)
  } {
      fmt.Println(v)
  }
  ```
- **HTTP routing improvements** — `http.ServeMux` now supports patterns with variables
- **`iter.Seq`** — built-in iterator sequences

### Migrating to Go 1.23

```bash
go mod edit -go=1.23
go mod tidy
go build ./...
```

---

## Go 1.22 (2024)

### New in Go 1.22

- **HTTP routing pattern variables** — `http.ServeMux` supports `/path/{var}`
- **`math/rand/v2`** — new v2 API
- **`go test -cpuprofile`** profiling improvements

---

## Go 1.21 (2023)

### New in Go 1.21

- **`log/slog`** — structured logging built-in, replaces third-party loggers
  ```go
  import "log/slog"
  
  slog.Info("hello", "key", "value", "count", 42)
  slog.Error("failed", "err", err)
  ```
- **Built-in `min`/`max`** — `min(1, 2, 3)`, `max(a, b)`
- **`clear()` function** — clears maps and slices
- **Conditional `go get`** — `go get example.com/foo@latest` updates to latest

---

## Version-Agnostic Best Practices (Always Apply)

### Always Use

- Go modules (`go.mod`, `go.sum`) — never GOPATH
- `context.Context` for all I/O operations
- Error wrapping with `%w`
- Structured logging (prefer `log/slog` on Go 1.21+, otherwise `zerolog`/`zap`)
- Explicit error handling — no `_` discards

### Go Module Best Practices

```bash
# Initialize module
go mod init github.com/myorg/myapp

# Add dependency
go get github.com/gin-gonic/gin@v1.11.0

# Tidy dependencies
go mod tidy

# View dependency graph
go mod graph

# Vendor dependencies
go mod vendor
```

### go.mod structure

```go
module github.com/myorg/myapp

go 1.24

require (
    github.com/gin-gonic/gin v1.11.0
    github.com/golang-jwt/jwt/v5 v5.2.0
    golang.org/x/crypto v0.31.0
)

require (
    github.com/bytedance/sonic v1.11.0 // indirect
    github.com/gabriel-vasile/mimetype v1.4.3 // indirect
    // ...
)
```

### Version Detection

```bash
# Check Go version
go version
# go version go1.24.0 linux/amd64

# Check Gin version
go list -m github.com/gin-gonic/gin
# github.com/gin-gonic/gin v1.11.0

# Check all deps versions
go list -m all
```

---

## Dependency Version Compatibility

### Gin v1.11.x (Latest)

| Dependency | Min Version | Notes |
|---|---|---|
| Go | 1.21+ (1.24 recommended) | |
| golang-jwt/jwt/**v5** | v5.x | **v4 is deprecated, always use v5** |
| gorm.io/gorm | v1.25+ | |
| go-redis/redis/v9 | v9.x | |
| bcrypt | golang.org/x/crypto | |

### Common Compatibility Issues

1. **jwt-go → jwt/v5** — `github.com/golang-jwt/jwt/v5` **not** `github.com/golang-jwt/jwt/v4` (v4 is deprecated)
2. **old Gin binding** — `binding:"required"` not `validate:""` on Gin v1.9+
3. **context typo** — `c.Request.Context()` not `c.Content` (that doesn't exist)

---

## Agent Checklist Per Task

Before working on any Go/Gin task:
- [ ] Identify Go version (`go version`)
- [ ] Check Gin version (`go list -m github.com/gin-gonic/gin`)
- [ ] Load version-specific rules above
- [ ] Check if feature is version-specific (slog, range-over-func, etc.)
- [ ] Apply version-specific patterns, not generic ones