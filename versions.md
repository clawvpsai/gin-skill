# Go/Gin Version-Specific Requirements

## Active Go Versions

- **Go 1.26** — Current stable (2026)
- **Go 1.25** — Previous stable
- **Go 1.24** — Still supported

## Version Selector Prompt

When working with Gin, check Go version first:
```bash
go version | grep -oP 'go\d+\.\d+'
```

Then load the relevant version sections below.

---

## Go 1.26 (Current — 2026)

### New in Go 1.26

- **Improved toolchain** — better error messages and diagnostics
- **Performance optimizations** — refined garbage collection
- **`go mod tidy`** improvements
- **Enhanced `go test`** — better test output

### Gin-Specific (Gin v1.12+)

- Use `gin.Context.Param()` — works with all Go versions
- `gin.H{}` is just `map[string]interface{}` — no changes needed

---

## Go 1.25 (2025)

### New in Go 1.25

- **`go mod graph`** improvements
- **Toolchain refinements**
- **Performance improvements across standard library**

---

## Go 1.24 (February 2025)

### New in Go 1.24

- **Generic type aliases** — type aliases may now be parameterized
  ```go
  type List[T any] = []T  // now valid!
  ```
- **Improved `go test -cpuprofile`** profiling
- **`math/rand/v2`** — new v2 API for random numbers
- **Toolchain improvements** — better `go mod` handling

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
- **`iter.Seq`** — built-in iterator sequences
- **HTTP routing pattern variables** — `http.ServeMux` supports `/path/{var}`

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

# Add dependency (latest Gin)
go get github.com/gin-gonic/gin@latest

# Tidy dependencies
go mod tidy

# View dependency graph
go mod graph
```

### go.mod structure

```go
module github.com/myorg/myapp

go 1.26

require (
    github.com/gin-gonic/gin v1.12.0
    github.com/golang-jwt/jwt/v5 v5.2.1
    golang.org/x/crypto v0.33.0
)
```

### Version Detection

```bash
# Check Go version
go version
# go version go1.26.2 linux/amd64

# Check Gin version
go list -m github.com/gin-gonic/gin
# github.com/gin-gonic/gin v1.12.x

# Check all deps versions
go list -m all
```

---

## Dependency Version Compatibility

### Gin v1.12.x (Latest)

| Dependency | Min Version | Notes |
|---|---|---|
| Go | 1.21+ (1.26 recommended) | |
| golang-jwt/jwt/**v5** | v5.x | **v4 is deprecated, always use v5** |
| gorm.io/gorm | v1.25+ | |
| go-redis/redis/v9 | v9.x | |
| bcrypt | golang.org/x/crypto | |

### Common Compatibility Issues

1. **jwt-go → jwt/v5** — `github.com/golang-jwt/jwt/v5` **not** `github.com/golang-jwt/jwt/v4` (v4 is deprecated)
2. **Binding tags** — `validate:""` not `binding:""` for Gin v1.9+
3. **context** — `c.Request.Context()` not `c.Content`

---

## Agent Checklist Per Task

Before working on any Go/Gin task:
- [ ] Identify Go version (`go version`)
- [ ] Check Gin version (`go list -m github.com/gin-gonic/gin`)
- [ ] Load version-specific rules above
- [ ] Check if feature is version-specific
- [ ] Apply version-specific patterns, not generic ones