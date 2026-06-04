# Go/Gin Version-Specific Requirements

## Active Go Versions

- **Go 1.26** — Current stable (go1.26.4, released 02 Jun 2026)
- **Go 1.25** — Previous stable (go1.25.11, released 02 Jun 2026)
- **Go 1.24** — Still supported (minimum for Gin v1.12)

## Version Selector Prompt

When working with Gin, check Go version first:
```bash
go version | grep -oP 'go\d+\.\d+'
```

Then load the relevant version sections below.

---

## Go 1.26 (Current — Feb 2026)

### New in Go 1.26

- **Green Tea Garbage Collector** — enabled by default in Go 1.26. Previously experimental in Go 1.25. Redesigned marking and scanning for small objects with better locality and CPU scalability. Expected 10–40% reduction in GC overhead for GC-heavy programs.
- **`go fix` overhaul** — completely rebuilt on the same analysis framework as `go vet`. Over 20 analyzers that automatically update code to newer language features and stdlib APIs. Run `go fix ./...` to modernize your codebase. Covers deprecated API migrations, internal packages, stdlib changes, and more.
- **Expression-based `new()`** — the built-in `new()` function now accepts an optional initial value expression:
  ```go
  p := new(int)        // same as before
  p := new(int, 42)    // new: initialize with expression
  ```
- **Self-referential generics** — type parameters can refer to themselves in their definition:
  ```go
  type Adder[T any] struct { sum T }
  func (a *Adder[T]) Add(v T) { a.sum += v }  // references Adder[T]
  ```
- **Experimental Goroutine Leak Profiler** — new `goroutineleak` profile type in `runtime/pprof`. Enable with `GOEXPERIMENT=goroutineleakprofile` at build time. Detects leaked goroutines in production.
- **`slices` and `maps` packages** — `maps.Copy()`, `maps.Clone()` for cleaner code
- **Improved toolchain** — better error messages and diagnostics

**Release:** Go 1.26.4 released 02 Jun 2026

### Go 1.26.4 Security Patch (2026-06-02)

- **Security fixes** to `crypto/x509`, `mime`, and `net/textproto` packages
- Bug fixes to compiler, runtime, `go fix` command, and `crypto/fips140` package
- **Upgrade immediately** if running Go 1.26.0–1.26.3 in production

---

## Go 1.27 (Release Freeze — Expected Aug 2026)

The Go 1.27 release freeze began **May 20, 2026** — 15 days in as of June 4, no RC1 yet. Monitor the [Go release dashboard](https://dev.golang.org/release) for tag announcements. The draft release notes page at [go.dev/doc/go1.27](https://go.dev/doc/go1.27) lists all confirmed features. RC1 expected soon. Expected final release: **August 2026**.

**For agents:** When RC1 drops, check the release notes for new stdlib/toolchain features before applying version-specific patterns. macOS 12 is dropped in Go 1.27 — use Go 1.26 for macOS 12 environments.

### New in Go 1.27

- **Generic methods** — a method declaration may declare its own type parameters. This widely anticipated change allows adding generic methods to any type (including interfaces, with some restrictions). Interface methods cannot be implemented by generic methods, and interface methods may not declare type parameters.
  ```go
  // Generic method on a concrete type
  type Stack[T any] struct { items []T }
  func (s *Stack[T]) PushAll(vals ...T) { s.items = append(s.items, vals...) }

  // Generic method on an interface
  type Stringer interface {
      String() string
      GenericMap[F any](func(T) F) []F  // generic method in interface
  }
  ```
- **Flexible struct literal keys** — a key in a struct literal may now be any valid field selector for the struct type, not just a top-level field name. Enables cleaner initialization patterns with nested or promoted fields.
  ```go
  type Inner struct { X int }
  type Outer struct { Inner }

  // Go 1.27: use dot notation as struct literal keys
  // o := Outer{Inner: {X: 1}}  // was required before
  ```
- **Native `uuid` package in stdlib** — `uuid` package provides `New()`, `NewRandomV7()`, and `Parse()`. `UUID` is `[16]byte` — API-compatible with `github.com/google/uuid`. UUIDv7 (time-ordered) is recommended for database primary keys. See [Go UUID Proposal](https://rednafi.com/shards/2026/04/go-uuid/).
  ```go
  import "uuid"

  id := uuid.New()           // UUIDv4 (random)
  id := uuid.NewRandomV7()  // UUIDv7 (time-ordered — better for DB indexes)
  parsed, err := uuid.Parse("550e8400-e29b-41d4-a716-446655440000")
  ```
- **`math/big.Int.Divide()`** — new method computes both quotient and remainder in a single call, matching the behavior of `big.Int.QuoRem` but with a simpler API.
  ```go
  var q, r big.Int
  r.Divide(&q, &r, 3)  // q = n/3, r = n%3
  ```
- **`net/url.URL.Clone()` and `net/url.Values.Clone()`** — deep-copy methods for URL and query parameter maps. Useful when modifying request URLs in middleware without affecting the original.
  ```go
  cloned := req.URL.Clone()       // deep copy of URL
  params := values.Clone()        // deep copy of query params
  ```
- **`go/scanner.Scanner.End()`** — new method returns the end position of the last token scanned. Useful for source code transformation tools and custom linters.
- **`math/rand/v2` generic `*Rand.N` method** — the `*Rand` type now has a generic `N` method matching the top-level `N` function, enabling per-RNG-source random selection with type-safe bounds.
- **`net.UnixConn` read methods return `io.EOF` directly** — previously wrapped in `net.OpError`; now returns `io.EOF` directly when the underlying read returns EOF. Quieter error handling for clean connection shutdowns.
- **QUIC/TLS support** — `tls.ConnectionState` now exposes QUIC connection state for HTTP/3 implementations using the stdlib TLS package directly.
- **macOS 12 dropped** — Go 1.26 is the last release supporting macOS 12. Upgrade to Go 1.26 or later for macOS development.
- **`go fix` improvements** — additional analyzers for deprecated patterns
- **Toolchain refinements** — continued improvement to error messages and diagnostics

**Release notes:** [go.dev/doc/go1.27](https://go.dev/doc/go1.27) (draft, finalized at release)
**Tracking:** [golang/go#76474](https://github.com/golang/go/issues/76474)

---

## Go 1.25 (Previous Stable — Aug 2025)

### New in Go 1.25

- **Container-aware GOMAXPROCS** — Go runtime now automatically detects cgroup CPU limits on Linux. In container environments (Kubernetes, Docker), `GOMAXPROCS` now defaults to the container's CPU limit rather than the number of logical CPUs. Also periodically updates `GOMAXPROCS` if container limits change. Disable with `GODEBUG=containermaxprocs=0` if needed. This is a significant performance improvement for cloud-native Go services — no more CPU throttling due to CPU limits exceeding actual allocation.
- **`testing/synctest` package** — new package for deterministic testing of concurrent code. Available since Go 1.24 under `GOEXPERIMENT=synctest`, now stable in Go 1.25. The `synctest.Test` function runs a test in an isolated goroutine world; `synctest.Wait` waits for all goroutines to finish. Use for testing concurrent code that would normally involve `time.Sleep` or arbitrary waits.
  ```go
  import "testing/synctest"

  func TestConcurrent(t *testing.T) {
      synctest.Test(func() {
          // all goroutine activity completes before this returns
          go func() { /* ... */ }()
      })
  }
  ```
- **Toolchain refinements**
- **Performance improvements across standard library**
- **Required for Gin v1.12.x** — minimum Go version raised from 1.18 to 1.24

**Release:** Go 1.25.11 released 02 Jun 2026

### Go 1.25.11 Security Patch (2026-06-02)

- **Security fixes** to `crypto/x509`, `mime`, and `net/textproto` packages (same CVEs as Go 1.26.4)
- Bug fixes to compiler and runtime
- **Upgrade immediately** if running Go 1.25.0–1.25.10 in production

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
- **Escaped Path Option** — `UseEscapedPath = true` on Engine enables escaped path routing (uses url.EscapedPath)
  ```go
  r := gin.New()
  r.UseEscapedPath = true // use url.EscapedPath() for routing
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

# Add golang.org/x/sync for errgroup/semaphore (used in concurrency patterns)
go get golang.org/x/sync@latest

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
    github.com/golang-jwt/jwt/v5 v5.3.1
    golang.org/x/crypto v0.52.0
    golang.org/x/sync v0.20.0
)
```

### Version Detection

```bash
# Check Go version
go version
# go version go1.26.4 linux/amd64

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
| go-redis/redis/v9 | v9.x | **Current: v9.20.0 (May 2026)** |
| golang.org/x/crypto | v0.52.0+ | Latest stable; Gin main uses v0.49.0 |
| golang.org/x/net | v0.55.0 | HTTP/2, TLS, DNS, HTTP trailers |
| golang.org/x/sys | v0.45.0 | System calls; comes with Go 1.26 toolchain |
| golang.org/x/text | v0.37.0 | Text encoding, Unicode |
| golang.org/x/arch | v0.27.0 | CPU architecture support |
| **golang.org/x/sync** | **v0.20.0** | **errgroup, semaphore — used in concurrency.md** |
| quic-go | v0.59.1 | HTTP/3 support (Gin v1.12+); **v0.59.1 released 2026-05-11**; v0.60 (dev) adds FIPS 140-3 support |
| mongo-driver | v2 | **BSON support upgraded to v2** |
| **gin-contrib/cors** | **v1.7.7** | **CORS middleware (github.com/gin-contrib/cors)** |
| **golang-migrate/migrate** | **v4.19.1** | **SQL migrations (github.com/golang-migrate/migrate)** |

### Common Compatibility Issues

1. **jwt-go → jwt/v5** — `github.com/golang-jwt/jwt/v5` **not** `github.com/golang-jwt/jwt/v4` (v4 is deprecated)
2. **Binding tags** — Gin uses `binding:""` (NOT `validate:""`) — Gin internally uses go-playground/validator but calls `SetTagName("binding")` to override the default tag name
3. **context** — `c.Request.Context()` not `c.Content`
4. **Go version too old** — Gin v1.12 requires Go 1.24+
5. **Dockerfile Go version** — Use `golang:1.26-alpine` or `golang:1.24-alpine`, NOT `golang:1.22-alpine`
6. **UUID package** — prefer stdlib `uuid` on Go 1.27+; fall back to `github.com/google/uuid` on Go <1.27

---

## Agent Checklist Per Task

Before working on any Go/Gin task:
- [ ] Identify Go version (`go version`)
- [ ] Check Gin version (`go list -m github.com/gin-gonic/gin`)
- [ ] Load version-specific rules above
- [ ] Check if feature is version-specific
- [ ] Apply version-specific patterns, not generic ones

---

## Updated from Research (2026-06-04)

### Go 1.27 Development Status
- Go 1.27 release freeze began **May 20, 2026** — **15 days in as of June 4**, no RC1 yet ([golang/go#76474](https://github.com/golang/go/issues/76474))
- Draft release notes at [go.dev/doc/go1.27](https://go.dev/doc/go1.27) confirm all features below
- **Go 1.26.4 and Go 1.25.11 released 2026-06-02** — security patches for `crypto/x509`, `mime`, `net/textproto`
- Expected final release: **August 2026**
- **macOS 12 dropped** — Go 1.26 is the last release supporting macOS 12

### Go 1.27 New Features (Confirmed from Draft Release Notes)
- **Generic methods** — method declarations may now declare type parameters. Major language change. Interfaces may include generic methods, but cannot be implemented by generic methods.
- **Flexible struct literal keys** — keys can be any valid field selector (e.g., nested/promoted field access)
- **Native `uuid` package** — stdlib `uuid.New()`, `uuid.NewRandomV7()`, `uuid.Parse()`; `[16]byte` type; API-compatible with `github.com/google/uuid`
- **`math/big.Int.Divide()`** — quotient + remainder in one call
- **`net/url.URL.Clone()` / `net/url.Values.Clone()`** — deep copy of URLs and query params
- **`go/scanner.Scanner.End()`** — get end position of last token
- **`math/rand/v2` generic `*Rand.N`** — per-RNG random selection
- **`net.UnixConn` EOF behavior** — returns `io.EOF` directly instead of wrapped in `net.OpError`
- **QUIC/TLS state** — `tls.ConnectionState` exposes QUIC connection state
- Gin v1.13 not yet released — v1.12.0 (Feb 2026) remains current

### Versions

- **Gin v1.12.0** — released 2026-02-28, current latest (no v1.13 yet)
- **go-redis v9.20.0** — released 2026-05-28, latest stable
- **GORM v1.31.1** — released 2025-11-02, latest stable
- **Go 1.26.4** — **current stable (security patch, 02 Jun 2026)**
- **Go 1.25.11** — **previous stable (security patch, 02 Jun 2026)**
- **Go 1.27** — in release freeze (15 days in as of Jun 4), no RC1 yet
- **golang-jwt/jwt v5.3.1** — released 2026-01-28, latest stable
- **golang.org/x/crypto v0.52.0** — latest stable
- **golang.org/x/net v0.55.0** — latest stable (HTTP/2, TLS, HTTP trailers)
- **golang.org/x/sys v0.45.0** — comes with Go 1.26 toolchain
- **golang.org/x/text v0.37.0** — latest stable
- **golang.org/x/arch v0.27.0** — latest stable
- **golang.org/x/sync v0.20.0** — errgroup, semaphore — used in concurrency.md patterns
- **quic-go v0.59.1** — HTTP/3 support (Gin v1.12+); released 2026-05-11
- **quic-go v0.60 (development)** — adds FIPS 140-3 support when built with Go 1.26+
- **gin-contrib/cors v1.7.7** — CORS middleware (github.com/gin-contrib/cors); released 2026-03-28
- **golang-migrate/migrate v4.19.1** — SQL migrations (github.com/golang-migrate/migrate, Nov 2025)

### Sources
- https://go.dev/doc/go1.27 (draft release notes — confirmed features)
- https://github.com/golang/go/issues/76474 (Go 1.27 tracking)
- https://github.com/gin-gonic/gin/releases
- https://github.com/redis/go-redis/releases
- https://github.com/go-gorm/gorm/releases
- https://github.com/quic-go/quic-go/releases
- https://github.com/gin-contrib/cors/releases
- https://github.com/golang-migrate/migrate/releases
- https://go.dev/dl/
- https://go.dev/doc/devel/release
- https://rednafi.com/shards/2026/04/go-uuid/
