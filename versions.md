# Go/Gin Version-Specific Requirements

## Active Go Versions

- **Go 1.26** — Current stable (go1.26.4, verified 2026-06-19 via go.dev/dl)
- **Go 1.25** — Previous stable (go1.25.11, verified 2026-06-19 via go.dev/dl)
- **Go 1.24** — Minimum for Gin v1.12

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
- **Experimental `simd/archsimd` package** — Go 1.26 introduces a new experimental `simd/archsimd` package providing low-level access to architecture-specific SIMD instructions (SSE/AVX on amd64, NEON on arm64). Enable with `GOEXPERIMENT=simd` at build time. Vector types are architecture-specific (e.g. `archsimd.Int32x4`). This is the architecture-tied lower layer; Go 1.27 adds a portable wrapper on top.
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
- **Goroutine Leak Profiler (experimental in 1.26, GA in 1.27)** — new `goroutineleak` profile type in `runtime/pprof`. In Go 1.26 enable with `GOEXPERIMENT=goroutineleakprofile` at build time. In Go 1.27+ the profile is **generally available** with no experiment flag, exposed at `net/http/pprof` endpoint `/debug/pprof/goroutineleak`. Detects leaked goroutines in production.
- **`slices` and `maps` packages** — `maps.Copy()`, `maps.Clone()` for cleaner code
- **Improved toolchain** — better error messages and diagnostics

**Release:** go1.26.4 — verified current as of 2026-06-19 against `go.dev/dl`

> ⚠️ **Verify latest before building:** `curl -s https://go.dev/dl/?mode=json | python3 -c "import sys,json; [print(p['version']) for p in json.load(sys.stdin)[:5]]"`

---

## Go 1.27 (Release Freeze — Expected Aug 2026)

The Go 1.27 release freeze began **May 20, 2026**. Monitor the [Go release dashboard](https://dev.golang.org/release) for RC announcements. The release notes page at [go.dev/doc/go1.27](https://go.dev/doc/go1.27) lists all confirmed features. RC1 expected soon. Expected final release: **August 2026**.

**For agents:** When RC1 drops, check the release notes for new stdlib/toolchain features before applying version-specific patterns. macOS 13 Ventura is required in Go 1.27 — use Go 1.26 for macOS 12 environments.

> **No Go 1.27 RC as of June 19, 2026** — Go 1.27 remains in release freeze with no RC1 tagged yet. Release freeze started May 20, 2026 (30 days in). Monitor [go.dev/dl](https://go.dev/dl/?mode=json) for RC1. Expected ~August 2026. Last re-verified 2026-06-19 12:10 UTC against go.dev/dl/?mode=json (still go1.26.4 / go1.25.11 stable — no RC1 yet).

### New in Go 1.27

- **Generic methods** — a method declaration may declare its own type parameters. This widely anticipated change allows adding generic methods to any type (including interfaces, with some restrictions). Interface methods cannot be implemented by generic methods, and interface methods may not declare type parameters.
- **Experimental `simd` package (portable wrapper)** — new `simd` package on top of `simd/archsimd` that is portable and vector-size-agnostic. Vector types like `simd.Int8s` and `simd.Float32s` use "scalable" operations that are hardware-supported or easily emulated across architectures and vector widths. Enable with `GOEXPERIMENT=simd` at build time. Useful for hot-path numeric and JSON-handling code in Gin services once stable. See the [Go 1.27 release notes draft](https://go.dev/doc/go1.27) for the design and roadmap.
- **Function type inference generalized (Go 1.27)** — function type inference now applies in **all** contexts where a generic function is assigned to a variable of (or converted to) a matching function type. Previously inference only worked in call expressions. Useful for storing generic function values in fields, slices, or returning them from factories.
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
- **Size-specialized memory allocation** — compiler now generates calls to size-specialized memory allocation routines, reducing cost of small (<80 byte) allocations by up to 30%. Expected ~1% overall improvement in allocation-heavy programs. Binary size increases ~60KB. Opt out with `GOEXPERIMENT=nosizespecializedmalloc` (temporary opt-out, removed in Go 1.28).
- **Toolchain refinements** — continued improvement to error messages and diagnostics

### New `encoding/json/v2` Package (Go 1.27)

Major redesign of `encoding/json` in a new `v2` sub-package — not a replacement, `encoding/json` v1 is fully supported:

```go
import "encoding/json/v2"

// New API with variadic Options
data, err := jsonv2.Marshal(v, jsonv2.OMitZero, jsonv2.Name("custom_key"))

// MarshalWrite writes directly to a Writer — no intermediate bytes.Buffer needed
err = jsonv2.MarshalWrite(w, v)

// UnmarshalRead reads directly from a Reader
err = jsonv2.UnmarshalRead(r, &v)

// Lower-level token decoding
err = jsonv2.UnmarshalDecode(dec, &v)
```

**`encoding/json/jsontext`** — lower-level streaming JSON token processing:
```go
import "encoding/json/jsontext"
dec := jsontext.NewDecoder(r)
tok, err := dec.Token() // get next JSON token
```

**Stricter defaults than v1:**
- Rejects invalid UTF-8 in JSON strings
- Rejects duplicate names within a JSON object

### `encoding/json` v1 is Now Backed by v2 in Go 1.27 (Critical)

As of Go 1.27, the `encoding/json` v1 package is **re-implemented on top of the v2 implementation**. Behavior is preserved (most code works unchanged), but error message text may differ. The net effect:

- **Existing Gin handlers using `c.JSON()` / `c.ShouldBindJSON()` automatically get the v2 performance boost** on Go 1.27+ — ~1.8× faster marshal, up to ~10× faster unmarshal, fewer allocations.
- The v1 `MarshalJSON` / `UnmarshalJSON` interface is still honored, but v2's `MarshalJSONTo` / `UnmarshalJSONFrom` (taking `*jsontext.Encoder` / `*jsontext.Decoder`) is now the preferred custom-type interface for hot paths.
- Behavior differences to watch: stricter UTF-8/duplicate-key rejection, slightly different number formatting, and a new `jsonv2.DiscardUnknownMembers` / `jsonv2.RejectDuplicateMembers` opt-in (v1 stays lenient by default).
- **Action for Gin projects:** just upgrade to Go 1.27 when it ships and existing code benefits — no migration required. For maximum throughput on JSON-heavy paths, register a custom `render.Render` to call `jsonv2.MarshalWrite` directly (see `responses.md` → "JSON v2 Custom Renderer").

> Source: [Go 1.27 Release Notes (draft)](https://go.dev/doc/go1.27) — *"The `encoding/json` package is now backed by the v2 implementation. Marshaling and unmarshaling behavior is preserved, but the exact text of error messages may differ."*

**Migration:** Use `encoding/json/v2` in new code. Existing `encoding/json` v1 code works unchanged — v1 is **not deprecated**.

**Performance (vs v1, measured on representative workloads):**
- **Unmarshal: 2.7–10.2× faster** — 2.7× on small payloads, 10.2× on large/structured JSON. Real-world: the k8s OpenAPI spec unmarshals **~40× faster** when using `UnmarshalJSONFrom` (the streaming custom-type method).
- **Marshal: ~1.8× faster on typical payloads** — close to v1 for tiny data, wins big for medium/large. Some datasets are still marginally slower; benchmark your shape.
- **Allocations: significant reduction** — many structs go to **zero heap allocations** for marshal; unmarshal cuts intermediate buffer churn.
- See benchmarks: [`github.com/go-json-experiment/jsonbench`](https://github.com/go-json-experiment/jsonbench).

**Streaming custom types — `MarshalJSONTo` / `UnmarshalJSONFrom`:**
```go
// v2 streaming interface — implement these on your types for O(n) instead of O(n²)
type MyType struct { /* ... */ }

func (m *MyType) MarshalJSONTo(w *jsontext.Encoder) error {
    // Write tokens directly to the encoder — no intermediate []byte
    if err := w.WriteToken(jsontext.BeginObject); err != nil { return err }
    // ... write tokens
    return w.WriteToken(jsontext.EndObject)
}

func (m *MyType) UnmarshalJSONFrom(dec *jsontext.Decoder) error {
    // Read tokens directly from decoder — no reflection per nested element
    // ...
    return nil
}
```
The streaming interface converts certain **O(n²) unmarshaling scenarios into O(n)**. Critical for large docs (k8s OpenAPI, OpenAPI specs, large config files).

**`OMitZero` (v2's stricter `omitempty`):**
- v2 introduces `OMitZero` option that omits fields with JSON-empty values (not just Go zero). For struct fields, this means a non-pointer field whose marshaled form is `{}` is omitted.
- Existing v1 `omitempty` tag is **backward-compatible** — still omits zero values for pointers, slices, maps, strings, numbers, bools. Struct fields are where behavior diverges.
- Use `OMitZero` option or `omitzero` struct tag for v2 semantics.

**Gin integration:** Gin's `c.JSON()` uses `encoding/json` v1. To use v2 in a Gin handler, register a custom `render.Render` (see `responses.md` → "JSON v2 Custom Renderer"). You can also swap the global JSON renderer via `engine.JSON = func(...)`.

**Pre-1.27 backport:** On Go 1.24/1.25/1.26, use the experiment `GOEXPERIMENT=jsonv2 go build`, or import [`github.com/go-json-experiment/json`](https://github.com/go-json-experiment/json) (the same API) into a vendor or replace directive in go.mod.

### Go 1.27 Post-Quantum Cryptography

- **`crypto/mldsa` package** — implements the post-quantum **ML-DSA (Module-Lattice-Based Digital Signature Algorithm)** specified in FIPS 204. Parameter sets: `MLDSA44`, `MLDSA65`, `MLDSA87` (trade off signature size vs speed). `crypto/x509` accepts ML-DSA keys/signatures; `crypto/tls` exposes them as `tls.MLDSA44`/`65`/`87` `SignatureScheme` values for TLS 1.3. The new `crypto.Hash` value `crypto.MLDSAMu` is used as a signaling mechanism for External μ (mu) ML-DSA signing.
- **`crypto/tls` MLKEM1024** — the `MLKEM1024` post-quantum key exchange is now supported. Enable by adding it to `tls.Config.CurvePreferences` (alongside or instead of the existing `X25519MLKEM768`). Use for HTTP/2 + TLS 1.3 servers in production to get hybrid post-quantum key exchange against capable clients.
- **Why it matters for Gin services:** forward-secrecy against future quantum attackers ("harvest now, decrypt later"). Enable both for any service that handles long-lived secrets (sessions, JWT signing keys, customer PII) — the overhead is small and interop with non-PQ clients is preserved.

### Go 1.27 Other New Standard Library APIs

- **`bytes.CutLast(s, sep) (before, after, found)`** — symmetric with `bytes.Cut`, slices around the **last** occurrence of a separator. Useful for parsing paths (`/api/v1/users` → `/api/v1` + `/users`), log lines, and filenames with a single call instead of `LastIndex` + manual slicing.
- **`net/url.URL.Clone()` / `net/url.Values.Clone()`** — deep copy of URLs and `url.Values` query params. Handlers can safely mutate cloned request URLs without affecting the original.
- **`crypto/x509.Certificate.RawSignatureAlgorithm`** (and `CertificateRequest.RawSignatureAlgorithm`, `RevocationList.RawSignatureAlgorithm`) — exposes the DER-encoded `AlgorithmIdentifier` even when the high-level `SignatureAlgorithm` field is `UnknownSignatureAlgorithm`. Use when verifying certificates signed with non-standard algorithms (e.g. ML-DSA above).

### Go 1.27 Tools Improvements (Draft Release Notes)

- **Response file support** — `@file` parsing now supported for `compile`, `link`, `asm`, `cgo`, `cover`, and `pack` tools. Arguments in response files support single/double quotes, escape sequences, and backslash line continuation. GCC-compatible format for build system interoperability.
- **`go test` stdversion vet check** — `go test` now runs the `stdversion` vet check by default. Reports when code uses stdlib symbols newer than the `go` directive in `go.mod` allows.
- **`go fix` new modernizers:** `atomictypes` (atomic type aliases), `embedlit` (embed directives), `slicesbackward` (backward `slices` iteration), `unsafefuncs` (unsafe function conversions)
- **`go fix` removed/renamed:** `fmtappendf` analyzer removed; `waitgroup` analyzer renamed to `waitgroupgo` to avoid ambiguity.
- **`go mod tidy` for go 1.27+ modules** — automatically merges duplicate `require` blocks into at most two blocks (direct + indirect). No more scattered duplicates.
- **Compiler `//line` relative paths** — `//line` and `/*line*/` directives now resolve relative filenames against the directory of the file containing the directive, matching `go/scanner` behavior.
- **`go` command + bzr removed** — support for the `bzr` version control system removed from `go get`/`go mod`; cannot fetch modules from bzr servers.
- **GODEBUG recognition** — `go` command now recognizes deprecated GODEBUG settings in `go.mod` (`go debug` entries) and `//go:debug` comments; accepts if set to final default value, fails if set to old value.

### Go 1.27 Ports Changes

- **Darwin (macOS)**: Go 1.27 requires **macOS 13 Ventura or later** — macOS 12 Monterey support is dropped (previously announced in Go 1.26 release notes). Use Go 1.26 for macOS 12 environments.
- **PowerPC 64-bit big-endian Linux (ppc64)**: Now uses ELFv2 ABI. Cgo, PIE, and external linking now supported. Requires Linux kernel 3.13+ (RHEL7 backported this support to its 3.10 kernel). For pure-Go binaries with cgo options, set `CGO_ENABLED=0`.

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

**Release:** go1.25.11 — verified current as of 2026-06-19 via go.dev/dl

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

### Gin v1.13 (Upcoming — Due June 30, 2026)

- **Milestone #28**: ~50% complete, due **June 30, 2026** (last updated ~June 12, 2026)
- Not yet released — v1.12.x remains current
- Check [github.com/gin-gonic/gin/milestone/28](https://github.com/gin-gonic/gin/milestone/28) for open issues
- **For agents**: When v1.13 drops, check release notes before applying version-specific patterns.

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
    golang.org/x/crypto v0.53.0
    golang.org/x/sync v0.21.0
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
| Go | 1.25+ | **Required for quic-go v0.60.0+** |
| **github.com/golang-jwt/jwt/v5** | v5.x | **v4 is deprecated, always use v5** |
| gorm.io/gorm | v1.25+ | **Current: v1.31+** |
| **github.com/redis/go-redis/v9** | v9.x | **Current: v9.20.1** |
| golang.org/x/crypto | v0.53.0+ | Latest stable; Gin main uses v0.49.0 |
| golang.org/x/net | v0.55.0 | HTTP/2, TLS, DNS, HTTP trailers |
| golang.org/x/sys | v0.46.0 | System calls; comes with Go 1.26 toolchain |
| golang.org/x/text | v0.38.0 | Text encoding, Unicode |
| golang.org/x/arch | v0.28.0 | CPU architecture support |
| **golang.org/x/sync** | **v0.21.0** | **errgroup, semaphore — used in concurrency.md** |
| quic-go | v0.60.0 | HTTP/3 support (Gin v1.12+); **Go 1.25+ required**; FIPS 140-3 ready (Go 1.26+) |
| mongo-driver | v2 | **BSON support upgraded to v2** |
| **gin-contrib/cors** | **v1.7.7** | **CORS middleware (github.com/gin-contrib/cors)** |
| **golang-migrate/migrate** | **v4.19.1** | **SQL migrations (github.com/golang-migrate/migrate)** |
| **jackc/pgx/v5** | **v5.10.0** | **PostgreSQL driver (CVE-2026-33816 fixed in v5.9.0+, CVSS 9.8)** |

### Common Compatibility Issues

1. **jwt-go → jwt/v5** — `github.com/golang-jwt/jwt/v5` **not** `github.com/golang-jwt/jwt/v4` (v4 is deprecated)
2. **Binding tags** — Gin uses `binding:""` (NOT `validate:""`) — Gin internally uses go-playground/validator but calls `SetTagName("binding")` to override the default tag name
3. **context** — `c.Request.Context()` not `c.Content`
4. **Go version too old** — Gin v1.12 requires Go 1.24+; **quic-go v0.60.0 requires Go 1.25+**
5. **Dockerfile Go version** — Use `golang:1.26-alpine` or `golang:1.24-alpine`, NOT `golang:1.22-alpine`
6. **UUID package** — prefer stdlib `uuid` on Go 1.27+; fall back to `github.com/google/uuid` on Go <1.27
7. **quic-go v0.60.0 + FIPS** — FIPS 140-3 support requires Go 1.26+; Go 1.25 minimum for quic-go v0.60.0 itself
8. **redis client module path** — use `github.com/redis/go-redis/v9` not `github.com/redis/redis/v9`
9. **pgx/v5 CVE-2026-33816** — if using jackc/pgx v5, upgrade to **v5.9.0+** (CVSS 9.8 critical memory-safety vulnerability); v5.10.0 patches both CVE-2026-33816 and CVE-2026-33815

---

## Agent Checklist Per Task

Before working on any Go/Gin task:
- [ ] Identify Go version (`go version`)
- [ ] Check Gin version (`go list -m github.com/gin-gonic/gin`)
- [ ] Load version-specific rules above
- [ ] Check if feature is version-specific
- [ ] Apply version-specific patterns, not generic ones

---

## Updated from Research (2026-06-15)

### ⚠️ CRITICAL: Always Verify Versions Against go.dev/dl

The Go release cycle does NOT always produce security patch releases on the dates previous research suggested. **Before citing any specific patch version (e.g., go1.26.5, go1.25.12), always verify against the official source:**

```bash
curl -s https://go.dev/dl/?mode=json | python3 -c "import sys,json; [print(p['version'], p.get('stable','')) for p in json.load(sys.stdin)[:10]]"
```

Previous research incorrectly stated go1.26.5 and go1.25.12 existed as security patches. **The authoritative go.dev/dl API (2026-06-15) confirms the actual latest stable versions are go1.26.4 and go1.25.11.** No security patches above those versions have been released.

### Go 1.27 Development Status
- Go 1.27 release freeze began **May 20, 2026** — **30 days in as of June 19**, no RC1 yet
- Release notes page at [go.dev/doc/go1.27](https://go.dev/doc/go1.27) confirmed all features below
- Expected final release: **August 2026**
- **macOS 13 Ventura required** — Go 1.27 drops macOS 12 Monterey; Go 1.26 is the last release for macOS 12

### Go 1.27 New Features (Confirmed from go.dev/doc/go1.27)
- **Generic methods** — method declarations may now declare type parameters. Major language change. Interfaces may include generic methods, but cannot be implemented by generic methods.
- **Flexible struct literal keys** — keys can be any valid field selector (e.g., nested/promoted field access)
- **Native `uuid` package** — stdlib `uuid.New()`, `uuid.NewRandomV7()`, `uuid.Parse()`; `[16]byte` type; API-compatible with `github.com/google/uuid`
- **`math/big.Int.Divide()`** — quotient + remainder in one call
- **`net/url.URL.Clone()` / `net/url.Values.Clone()`** — deep copy of URLs and query params
- **`go/scanner.Scanner.End()`** — get end position of last token
- **`math/rand/v2` generic `*Rand.N`** — per-RNG random selection
- **`net.UnixConn` EOF behavior** — returns `io.EOF` directly instead of wrapped in `net.OpError`
- **QUIC/TLS state** — `tls.ConnectionState` exposes QUIC connection state
- **Faster memory allocation** — size-specialized malloc reduces small (<80B) allocation cost by up to 30%, ~1% overall improvement. Binary size +60KB. Opt-out via `GOEXPERIMENT=nosizespecializedmalloc` (temporary).

### Go 1.27 Tools (Confirmed from go.dev/doc/go1.27)
- Response file (`@file`) support for compile/link/asm/cgo/cover/pack — GCC-compatible
- `go test` now runs `stdversion` vet check by default
- `go fix` new modernizers: `atomictypes`, `embedlit`, `slicesbackward`, `unsafefuncs`
- `go fix`: `fmtappendf` removed, `waitgroup` renamed to `waitgroupgo`
- `go mod tidy` auto-merges duplicate `require` blocks for `go 1.27+` modules
- Compiler `//line` relative path resolution improved
- `bzr` VCS support removed from `go` command

### Go 1.27 Ports
- **Darwin**: macOS 13 Ventura or later required — macOS 12 dropped
- **PowerPC ppc64 big-endian Linux**: ELFv2 ABI switch, Cgo/PIE/external linking now supported; kernel 3.13+ required

### Gin v1.13
- Milestone #28: **~55% complete (15/27 issues closed)**, due **June 30, 2026** (last updated 2026-06-18)
- Check [github.com/gin-gonic/gin/milestone/28](https://github.com/gin-gonic/gin/milestone/28) for open issues
- v1.12.x remains current until v1.13 ships

### New in Go 1.27: `encoding/json/v2`
- Major redesign with `MarshalWrite`, `UnmarshalRead`, `UnmarshalDecode`, variadic Options, stricter UTF-8/duplicate-key defaults
- `encoding/json/jsontext` for streaming token-level JSON processing
- v1 is **not deprecated** — existing code works unchanged

### Verified Versions (2026-06-19 — go.dev/dl API)

- **Gin v1.12.0** — released 2026-02-28, current latest (GitHub API confirmed)
- **Gin v1.13** — milestone #28, due 2026-06-30, **~55% complete (15/27 closed, 12 open)**, not yet released (verified 2026-06-19)
- **Go 1.26.4** — **current stable** (verified via go.dev/dl)
- **Go 1.25.11** — **previous stable** (verified via go.dev/dl)
- **Go 1.27** — in release freeze (**27 days as of Jun 16**), no RC1 yet, expected August 2026
- **go-redis v9.20.1** — latest stable
- **GORM v1.31.1** — latest stable
- **jackc/pgx v5 v5.10.0** — latest stable (Jun 3, 2026); **CVE-2026-33816 fixed in v5.9.0+** (critical memory-safety, CVSS 9.8); CVE-2026-33815 also patched in v5.10.0
- **golang-jwt/jwt v5.3.1** — latest stable
- **golang.org/x/crypto v0.53.0** — latest stable
- **golang.org/x/net v0.55.0** — latest stable
- **golang.org/x/sys v0.46.0** — comes with Go 1.26 toolchain
- **golang.org/x/text v0.38.0** — latest stable
- **golang.org/x/arch v0.28.0** — latest stable
- **golang.org/x/sync v0.21.0** — errgroup, semaphore — used in concurrency.md patterns
- **quic-go v0.60.0** — HTTP/3 support (Gin v1.12+); Go 1.25+ required; FIPS 140-3 ready (Go 1.26+)
- **gin-contrib/cors v1.7.7** — CORS middleware
- **golang-migrate/migrate v4.19.1** — SQL migrations

### Sources
- https://go.dev/dl/?mode=json (authoritative — verified 2026-06-19)
- https://github.com/gin-gonic/gin/releases (authoritative — verified 2026-06-19)
- https://github.com/gin-gonic/gin/milestones (Gin v1.13 milestone progress, verified 2026-06-19)
- https://go.dev/doc/go1.27 (Go 1.27 release notes)
- https://github.com/golang/go/issues/76474 (Go 1.27 tracking)
- https://dev.golang.org/release (Go release dashboard)
- https://github.com/gin-gonic/gin/milestone/28 (Gin v1.13 milestone)
