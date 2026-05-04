# Go/Gin Version-Specific Requirements

## Active Go Versions

- **Go 1.26** — Current stable (go1.26.2, May 2026)
- **Go 1.25** — Previous stable (go1.25.9)
- **Go 1.24** — Still supported (minimum for Gin v1.12)

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
- **Green Tea Garbage Collector** — enabled by default in Go 1.26. Previously experimental in Go 1.25. Redesigned marking and scanning for small objects with better locality and CPU scalability. Expected 10–40% reduction in GC overhead for GC-heavy programs.
- **Experimental Goroutine Leak Profiler** — new `goroutineleak` profile type in `runtime/pprof`. Enable with `GOEXPERIMENT=goroutineleakprofile`. Also available at `/debug/pprof/goroutine` when enabled. Detects leaked goroutines in production.
- **`go mod tidy`** improvements
- **Enhanced `go test`** — better test output
- **`slices` and `maps` packages** — `maps.Copy()`, `maps.Clone()` for cleaner code
- **Range over integers** — `for i := range n` now works (Go 1.23+)
- **Improved `build` package** — better cross-compilation support

---


## Go 1.25 (Previous Stable)

### New in Go 1.25

- **`go mod graph`** improvements
- **Toolchain refinements**
- **Performance improvements across standard library**
- **Required for Gin v1.12.x** — minimum Go version raised from 1.18 to 1.24

---

## Go 1.24 (February 2025 — Minimum for Gin v1.12)

### New in Go 1.24

- **Generic type aliases** — type aliases may now be parameterized
  ```go
  type List[T any] = []T  // now valid!
  ```
- **Improved `go test -cpuprofile`** profiling
- **`math/rand/v2`** — new v2 API for random numbers
- **Toolchain improvements** — better `go mod` handling
- **Minimum Go version for Gin v1.12** — all Gin v1.12+ projects must use Go 1.24+

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

## Gin v1.12.x (Current — February 2026)

### What's New in Gin v1.12

- **BSON Protocol Support** — render layer now supports BSON encoding
  ```go
  import "go.mongodb.org/mongo-driver/bson"
  c.Data(200, "application/bson", bsonData)
  ```
- **New Context Methods:**
  - `GetError()` / `GetErrorSlice()` — type-safe error retrieval from context
  - `Delete(key)` — remove keys from context
- **Flexible Binding** — URI and query binding now honor `encoding.UnmarshalText`
- **Escaped Path Option** — `UseRouters()` enables escaped/raw request path routing
  ```go
  r := gin.New()
  r.UseRouters() // use raw request path for routing
  ```
- **ProtoBuf Content Negotiation** — `c.ProtoBuf()` for gRPC-style responses
  ```go
  c.ProtoBuf(200, &Response{Success: true, Message: "ok"})
  ```
- **Colorized Latency in Logger** — easier to spot slow requests
- **Flush Safety** — `Flush()` no longer panics if `http.ResponseWriter` doesn't implement `http.Flusher`
- **Unix Socket Trust** — `X-Forwarded-For` always trusted over Unix sockets
- **Router Tree Optimizations** — fewer allocations, faster path parsing

### Gin v1.11 (HTTP/3 Support)

- **HTTP/3 Support** — Gin now supports HTTP/3 via QUIC
- **Form Improvements** — better form binding edge cases

### Gin v1.10

- Zero allocation router improvements
- Better error messages

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
    golang.org/x/crypto v0.45.0
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
| Go | 1.24+ | **Minimum raised to 1.24 in Gin v1.12** |
| golang-jwt/jwt/**v5** | v5.x | **v4 is deprecated, always use v5** |
| gorm.io/gorm | v1.25+ | **Current: v1.31+ (Nov 2025)** |
| go-redis/redis/v9 | v9.x | **Current: v9.19.0 (Apr 2026)** |
| golang.org/x/crypto | v0.45.0+ | Updated in Gin v1.12 |
| mongo-driver | v2 | **BSON support upgraded to v2** |
| quic-go | v0.57.1 | HTTP/3 support (Gin v1.11+) |

### Common Compatibility Issues

1. **jwt-go → jwt/v5** — `github.com/golang-jwt/jwt/v5` **not** `github.com/golang-jwt/jwt/v4` (v4 is deprecated)
2. **Binding tags** — `validate:""` not `binding:""` for Gin v1.9+
3. **context** — `c.Request.Context()` not `c.Content`
4. **Go version too old** — Gin v1.12 requires Go 1.24+
5. **Dockerfile Go version** — Use `golang:1.26-alpine` or `golang:1.24-alpine`, NOT `golang:1.22-alpine`

---

## Agent Checklist Per Task

Before working on any Go/Gin task:
- [ ] Identify Go version (`go version`)
- [ ] Check Gin version (`go list -m github.com/gin-gonic/gin`)
- [ ] Load version-specific rules above
- [ ] Check if feature is version-specific
- [ ] Apply version-specific patterns, not generic ones

---

## Updated from Research (2026-05)

### Versions

- **Gin v1.12.0** — released 2026-02-28, current latest
- **go-redis v9.19.0** — released 2026-04-28, latest stable
- **GORM v1.31.1** — released 2025-11-02, latest stable
- **Go 1.26.2** — current stable (no Go 1.27 yet as of May 2026)
- **Gin contributors** now managed via GitHub Contributors page (AUTHORS.md removed)

### Sources
- https://github.com/gin-gonic/gin/releases
- https://github.com/redis/go-redis/releases
- https://github.com/go-gorm/gorm/releases
- https://go.dev/dl/