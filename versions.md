# Go/Gin Version-Specific Requirements

## Active Go Versions

- **Go 1.26** — Current stable (go1.26.4, verified 2026-06-25 18:09 UTC via go.dev/dl). **Security patch go1.26.5 is imminent** — **8 pending CLs** on release-branch.go1.26 per dev.golang.org/release (snapshot 2026-06-25 18:09 UTC; was 7 on 2026-06-25 12:07 UTC, was 7 on 2026-06-23, was 6 on 2026-06-22 12:04 UTC, was 7 on 2026-06-21).
- **Go 1.25** — Previous stable (go1.25.11, verified 2026-06-25 18:09 UTC via go.dev/dl). **Security patch go1.25.12 is imminent** — 4 pending CLs on release-branch.go1.25 per dev.golang.org/release (snapshot 2026-06-25 18:09 UTC; was 4 on 2026-06-23, was 3 on 2026-06-22 12:04 UTC, was 4 on 2026-06-21, was 3 on 2026-06-17 — the `crypto/tls` FIPS backport that appeared on 2026-06-21 has since been pulled from the release dashboard; a new `cmd/compile` heap-allocated CL appeared in its place on 2026-06-23).
- **Go 1.24** — Minimum for Gin v1.12 (still security-supported until Go 1.26 + 1 stable)

## Pending Security Patches (2026-06-25)

The [dev.golang.org/release](https://dev.golang.org/release) dashboard (re-checked 2026-06-25 18:09 UTC) shows pending CLs for two upcoming security patch releases:

- **Go 1.26.5** — **8 pending CLs** across `cmd/compile`, `cmd/fix`, `cmd/link`, `runtime`, `security`, `x/tools/go/analysis`. The `crypto/tls` FIPS+`InsecureSkipVerify` CL that appeared on the dashboard on 2026-06-21 is **no longer listed** as of this snapshot. A new `cmd/compile` CL appeared on 2026-06-23 and a new `cmd/link` CL appeared on 2026-06-25. Highlights of the remaining CLs:
  - `security: fix CVE-2026-39822 [1.26 backport]` (embargoed runtime fix)
  - `runtime: version parsing fails [1.26 backport]` — fixes `runtime.Version()` parser edge case (e.g. on pseudo-versions with dirty/suffix tags)
  - `cmd/compile: prove misscompilation in slicemask folding leaves garbage in the upper half of the 32bits of the register when slicing a slice by a non constant value that is < cap [1.26 backport]`
  - `cmd/fix, x/tools/go/analysis: fix can print "applied 8 of 10 fixes; 2 files updated" and fail without actually applying any fixes [1.26 backport]`
  - `runtime: concurrent map read and map write with sync.RWMutex on ppc64le [1.26 backport]` (was already pending — pulled into the highlight section now that the FIPS CL was removed)
  - `cmd/compile: internal compiler error invalid heap allocated var without Heapaddr [1.26 backport]` (NEW as of 2026-06-23 00:11 UTC snapshot)
  - `cmd/link: peCreateExportFile generates invalid .def file when output name has trailing dot (c-shared) [1.26 backport]` (**NEW** as of 2026-06-25 18:09 UTC snapshot) — Windows PE linker fix for cross-compiled `-buildmode=c-shared` DLLs with trailing-dot output names. Affects only Windows DLL/plugin/c-shared builds; standard `.exe` and `.dll` (non-c-shared) builds are not affected.
- **Go 1.25.12** — 4 pending CLs across `cmd/compile`, `runtime`, `security`. The `crypto/tls` FIPS+`InsecureSkipVerify` CL that appeared on 2026-06-21 is no longer listed (same as 1.26.5 above); a new `cmd/compile` heap-allocated CL appeared in its place on 2026-06-23. Highlights:
  - `security: fix CVE-2026-39822 [1.25 backport]`
  - `runtime: concurrent map read and map write with sync.RWMutex on ppc64le [1.25 backport]`
  - `cmd/compile: prove misscompilation in slicemask folding [...] [1.25 backport]`
  - `cmd/compile: internal compiler error invalid heap allocated var without Heapaddr [1.25 backport]` (**NEW** as of 2026-06-23 00:11 UTC snapshot)

**CVE-2026-39822** (embargoed, per Go security policy): runtime-level security fix. Details under embargo until release announcement. Track [github.com/golang/go/issues/79005](https://github.com/golang/go/issues/79005).

**Action:** Re-run `curl -s https://go.dev/dl/?mode=json` before deploying production builds — the patches may have shipped since this snapshot. Track [dev.golang.org/release](https://dev.golang.org/release) for the announcement thread on [golang-announce](https://groups.google.com/g/golang-announce). Note that the `crypto/tls` FIPS+`InsecureSkipVerify` CL that previously appeared on the dashboard has since been pulled (likely reverted for further work); if your Gin service terminates TLS in FIPS 140-3 mode, monitor for the fix in a later patch release (Go 1.25.13 / 1.26.6).

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

**Release:** go1.26.4 — re-verified 2026-06-22 12:04 UTC against `go.dev/dl/?mode=json` (still current)

> ⚠️ **Verify latest before building:** `curl -s https://go.dev/dl/?mode=json | python3 -c "import sys,json; [print(p['version']) for p in json.load(sys.stdin)[:5]]"`

### Go 1.26 Breaking Changes Affecting Gin Handlers

These changes are observable when upgrading services from Go 1.25 → 1.26 and may surface in production. Watch for them in CI and staged rollouts.

- **`net/url.Parse` rejects malformed URLs with extra colons in the host** (e.g. `http://::1/`, `http://localhost:80:80/`, `http://localhost:8080:80/`) — RFC 3986 allows a single colon between hostname and port, so multiple colons are unambiguously a parse error. **Affects any Gin handler that calls `url.Parse` on a user-supplied string** (OAuth/OIDC redirect URI validation, webhook URL intake, deep-link parsing, well-known endpoint discovery). On Go 1.25 these parsed as malformed hosts; on Go 1.26 they return `*url.Error` immediately. Re-enable the old behavior with `GODEBUG=urlstrictcolon=0` if your service must accept the historical inputs during a migration. See [go.dev/issue/75223](https://github.com/golang/go/issues/75223).
  ```go
  // Recommended: validate with ParseRequestURI for redirect/callback URLs
  u, err := url.ParseRequestURI(redirect)
  if err != nil || u.Host == "" {
      c.JSON(400, gin.H{"error": "invalid redirect_uri"})
      return
  }
  // Then check that u.Host has exactly one colon (or no colon if no port)
  ```
- **Hybrid post-quantum TLS key exchanges enabled by default** — `SecP256r1MLKEM768` and `SecP384r1MLKEM1024` are now in `tls.Config.CurvePreferences` by default. Most clients will negotiate successfully; legacy or specialized clients (some older TLS terminators, certain proxies, embedded MQTT brokers) may fail TLS handshake. Test against your real client fleet before enabling in production. Disable with `GODEBUG=tlsmlkem=0` for one release cycle if you need a rollback. Affects Gin services that terminate TLS via `http.Server.TLSConfig` or via a reverse proxy that does its own TLS.
- **Green Tea GC enabled by default** — improved GC throughput but a small benchmark delta (10–40% GC overhead reduction on small-object workloads). Generally positive; benchmark your service to confirm. No action required.
- **`go test` vet now runs `stringer` check by default** — flags types with a `String()` method that don't satisfy `fmt.Stringer` interface correctly (e.g. pointer vs value receiver mismatches). Existing test suites that compile fine on Go 1.25 may start failing `go vet` / `go test` on Go 1.26. Run `go test ./...` in CI before upgrading to surface these.
- **`url.Parse` no longer accepts `localhost:8080:80` (multiple colons)** — same fix as above; both `Parse` and `ParseRequestURI` are affected. Source: [Go 1.26 release notes](https://go.dev/doc/go1.26).


---

## Go 1.27 (RC1 Released 2026-06-18 — Final Expected Aug 2026)

The Go 1.27 release freeze began **May 20, 2026**. Monitor the [Go release dashboard](https://dev.golang.org/release) for RC announcements. The release notes page at [go.dev/doc/go1.27](https://go.dev/doc/go1.27) lists all confirmed features. RC1 expected soon. Expected final release: **August 2026**.

**For agents:** When RC1 drops, check the release notes for new stdlib/toolchain features before applying version-specific patterns. macOS 13 Ventura is required in Go 1.27 — use Go 1.26 for macOS 12 environments.

> **Go 1.27 RC1 RELEASED 2026-06-18** — `go1.27rc1` shipped June 18, 2026 (verified 2026-06-25 00:15 UTC via `https://go.dev/dl/?mode=json` returning `go1.27rc1 stable=False`). Release notes: [go.dev/doc/go1.27](https://go.dev/doc/go1.27). The freeze-day count is now superseded by RC-phase: RC1 (Jun 18) → expected RC2 ~3 weeks later → final ~August 2026. **Do NOT run RC1 in production** — `go install golang.org/dl/go1.27rc1@latest && go1.27rc1 download && go1.27rc1 test ./...` to validate test suites against it. Many breaking/toolchain changes that affect Gin projects are listed below; see the [Auto-update 2026-06-25 00:15 UTC] section near the bottom of this file for the per-change callouts. Last re-verified 2026-06-25 00:15 UTC.

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
- **`crypto/x509.SystemCertPool` + `SSL_CERT_FILE` / `SSL_CERT_DIR` (Windows + Darwin)** — `SystemCertPool` now respects `SSL_CERT_FILE` and `SSL_CERT_DIR` env vars on Windows and Darwin. When set, roots are loaded from disk and the **native Go verifier** is used instead of the platform cert API. Can be disabled with `GODEBUG=x509sslcertoverrideplatform=0`. **Gin impact**: if you set these env vars in production (common in container deployments to inject a custom CA bundle), the platform trust store is replaced by the file/dir contents — `crypto/tls` connections will start succeeding/failing based on what's in those files, not the OS root store. Audit container images for accidental `SSL_CERT_FILE` leakage from base layers.
- **`crypto/tls.ConnectionState.LocalCertificate`** — new field exposing the certificate chain **presented to the connection peer** during the handshake. Pairs with the existing `PeerCertificates` (what the server received from the client). For Gin servers: use this to log/audit the exact chain the client saw (helpful when debugging "client says my cert is wrong" reports — they may be looking at the leaf, the intermediate, or the root you presented).
- **`crypto/tls.QUICConfig.ClientHelloInfoConn`** — new field for QUIC server handshakes that lets the TLS config know which `net.Conn` to use for `ClientHelloInfo.Conn` (the listener side). Relevant if you run Gin over HTTP/3 via a custom `quic.Listener` — until now the `Conn` field in the `ClientHelloInfo` was nil for QUIC server handshakes.
- **`crypto/tls.Config.Rand` deprecated** — for deterministic testing of TLS code paths, use `testing/cryptotest.SetGlobalRandom` (in the new `testing/cryptotest` package). Existing `Config.Rand` users will get a deprecation warning at compile time; plan to migrate before Go 1.30.
- **`crypto/ecdsa.PrivateKey.Sign` hash length check** — `PrivateKey.Sign` now verifies that the hash length matches the curve's expected output size when a non-nil `SignerOpts` is supplied. Previously a wrong-length hash would silently produce a bad signature that failed remote verification; now it errors at sign time. Tightens an interop footgun.
- **`database/sql.ConvertAssign` + `database/sql/driver.RowsColumnScanner`** — `ConvertAssign` exposes the type-conversion logic used by `Rows.Scan` so custom drivers can reuse it. The new optional `RowsColumnScanner` interface lets drivers scan directly into user-provided destinations (skipping the `interface{}` round-trip). Relevant for Gin projects that embed a custom driver (e.g. SQLite shim, TimescaleDB extension, in-memory test double).
- **`go/constant.StringLen`** — returns the length of a string `Value` without fully constructing the `Value`. Useful for static-analysis tools built on top of Gin (e.g. custom linters, request validators, OpenAPI generators that parse Go source).
- **`go/token.File.String()`** — small QoL: the `File` type now has a `String()` method, so it can be printed directly in error messages and `%v` formatting. Nice for code-gen tools that surface token positions.
- **`compress/flate` speedup — output bytes may differ** — Go 1.27 ships a faster DEFLATE encoder. The encoded output of `compress/flate.Writer` is **not byte-identical** to Go 1.26 (different implementation, same format). Because DEFLATE is the underlying compression for `archive/zip`, `compress/gzip`, `compress/zlib`, and `image/png`, the **outputs of those packages may also differ** from Go 1.26. **Gin impact**: any code that ships pre-compressed assets and asserts byte-equality of recompression (e.g. golden-file tests for `c.File("foo.zip")` middleware) will need a re-baseline. Also, if you cache `ETag` headers keyed on compressed-byte-length, the length may shift. Standard HTTP `Content-Encoding: gzip` clients tolerate this transparently — only your own test fixtures and cache keys are at risk.

### Go 1.27 Tools Improvements (Draft Release Notes)

- **Response file support** — `@file` parsing now supported for `compile`, `link`, `asm`, `cgo`, `cover`, and `pack` tools. Arguments in response files support single/double quotes, escape sequences, and backslash line continuation. GCC-compatible format for build system interoperability.
- **`go test` stdversion vet check** — `go test` now runs the `stdversion` vet check by default. Reports when code uses stdlib symbols newer than the `go` directive in `go.mod` allows.
- **`go fix` new modernizers:** `atomictypes` (atomic type aliases), `embedlit` (embed directives), `slicesbackward` (backward `slices` iteration), `unsafefuncs` (unsafe function conversions)
- **`go fix` removed/renamed:** `fmtappendf` analyzer removed; `waitgroup` analyzer renamed to `waitgroupgo` to avoid ambiguity.
- **`go mod tidy` for go 1.27+ modules** — automatically merges duplicate `require` blocks into at most two blocks (direct + indirect). No more scattered duplicates.
- **Compiler `//line` relative paths** — `//line` and `/*line*/` directives now resolve relative filenames against the directory of the file containing the directive, matching `go/scanner` behavior.
- **`go` command + bzr removed** — support for the `bzr` version control system removed from `go get`/`go mod`; cannot fetch modules from bzr servers.
- **GODEBUG recognition** — `go` command now recognizes deprecated GODEBUG settings in `go.mod` (`go debug` entries) and `//go:debug` comments; accepts if set to final default value, fails if set to old value.
- **`go doc package@version`** — `go doc` now supports `package@version` syntax (e.g. `go doc example.com/pkg@v1.2.3`) to view docs for a specific module version. Use during code review to check what changed in a dependency before bumping.
- **`go doc -ex`** — new flag lists executable examples (`Example*` functions in `_test.go` files) of the given package or symbol. Quick way to surface working code snippets for any package — handy when exploring new stdlib APIs (e.g. `go doc -ex uuid`).
- **`go test -json` `OutputType` field** — `go test -json` annotates `"Action":"output"` lines with an `"OutputType"` field (`"error"`, `"error-continue"`, or `"frame"`). Test runners and IDE plugins that parse test2json output should learn to consume this field for cleaner separation of test failures from test progress logs.
- **`go tool trace -http=:PORT` now localhost-only by default** — passing only a port (e.g. `-http=:6060`) now restricts the listener to `localhost`, matching `go tool pprof -http` behavior. To listen on all interfaces, pass the address explicitly (`-http=0.0.0.0:6060`). **Gin impact**: in container/VM dev environments where you remote-port-forward to read the trace UI, you must now specify the bind address — otherwise the trace UI will only be reachable from inside the container.
- **Linker `macos` / `macsdk` flags (macOS only)** — `go build` on macOS now accepts `-ldflags=-macos=<version> -ldflags=-macsdk=<version>` to set the `LC_BUILD_VERSION` load command's minimum-OS and SDK-version fields. Defaults to oldest supported macOS (13.0.0) and a recent SDK (26.2.0). Only relevant for cross-compiled Gin binaries for macOS distribution.
- **New `go` command: `asynctimerchan` GODEBUG setting REMOVED permanently** — the `GODEBUG=asynctimerchan=1` setting (added in Go 1.23 to allow old buffered `time.Timer`/`time.Ticker` channels) is **gone** in Go 1.27. Channels from the `time` package are **always unbuffered (synchronous)** now, regardless of any GODEBUG. **Gin impact**: any code that relies on the buffered-channel semantics for `time.After` (e.g. select-with-default patterns that assumed non-blocking send to a buffered timer channel) will need to be re-checked. The buffer-1 size that Go 1.22 used is fully retired.
- **New `go` command: TLS GODEBUG settings REMOVED permanently** — `tlsunsafeekm`, `tlsrsakex`, `tls3des`, `tls10server` (added in Go 1.22), and `x509keypairleaf` (added in Go 1.23) are all **gone** in Go 1.27. These were temporary escape hatches for re-enabling unsafe TLS behaviors — their removal is a security win. If any of your services, load balancers, orginy proxy chains rely on these settings to talk to a legacy peer, they will need to be fixed before upgrading to Go 1.27. **Gin impact**: search your codebase + config for `GODEBUG` settings — most likely you're not using any of these, but check.
- **`runtime/pprof` goroutine labels in tracebacks (Go 1.27+ modules)** — tracebacks for modules with `go` directives configuring Go 1.27 or later now include `runtime/pprof` goroutine labels in the header line. Helps in production debugging — labels set via `pprof.SetGoroutineLabels` will now appear in the stack header. Opt out with `GODEBUG=tracebacklabels=0` (the setting is expected to be kept indefinitely in case labels carry sensitive data).

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

**Release:** go1.25.11 — re-verified 2026-06-21 12:04 UTC against go.dev/dl/?mode=json (still current)

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
| **github.com/redis/go-redis/v9** | v9.x | **Current: v9.21.0** (2026-06-22, supersedes v9.20.1) |
| golang.org/x/crypto | v0.53.0+ | Latest stable; **Gin v1.13 (master, post PR #4707) will pull v0.52.0 transitively via validator v10.30.3 — BELOW floor; EXPLICITLY require v0.53.0+ in your go.mod when adopting Gin v1.13** |
| golang.org/x/net | v0.55.0 | HTTP/2, TLS, DNS, HTTP trailers |
| golang.org/x/sys | v0.46.0 | System calls; comes with Go 1.26 toolchain; **Gin v1.13 (master, post PR #4707) will pull v0.45.0 transitively via validator v10.30.3 — BELOW floor; EXPLICITLY require v0.46.0+ in your go.mod when adopting Gin v1.13** |
| golang.org/x/text | v0.38.0 | Text encoding, Unicode |
| golang.org/x/arch | v0.28.0 | CPU architecture support |
| **golang.org/x/sync** | **v0.21.0** | **errgroup, semaphore — used in concurrency.md** |
| quic-go | v0.60.0 | HTTP/3 support (Gin v1.12+); **Go 1.25+ required**; FIPS 140-3 ready (Go 1.26+) |
| mongo-driver | v2 | **BSON support upgraded to v2** |
| **gin-contrib/cors** | **v1.7.7** | **CORS middleware (github.com/gin-contrib/cors)** |
| **golang-migrate/migrate** | **v4.19.1** | **SQL migrations (github.com/golang-migrate/migrate)** |
| **pressly/goose** | **v3.27.1** | **SQL + Go migrations (github.com/pressly/goose/v3); requires Go 1.25+** |
| **ariga/atlas** | **v1.2.0** | **Declarative schema-as-code + versioned auto-planning (ariga.io/atlas-go-sdk/atlasexec)** |
| **squaredup/squawk** | **latest** | **PostgreSQL migration linter — CI guard for dangerous DDL** |
| **jackc/pgx/v5** | **v5.10.0** | **PostgreSQL driver (CVE-2026-33816 fixed in v5.9.0+, CVSS 9.8)** |
| **go-playground/validator/v10** | **v10.30.3** | **Gin v1.13 transitive (PR #4707 merged 2026-06-23 12:08 UTC, security-driven: "fixes multiple critical vulnerabilities from golang.org/x/crypto"). Validator v10.30.3's go.mod pulls x/crypto v0.52.0 + x/text v0.37.0 + x/sys v0.45.0 — see Floor-Piercing Risk section below. Released 2026-05-29 (skipped v10.30.2)** |

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

## Updated from Research (2026-06-21)

### New CVEs Affecting Gin/Go (June 2026)

- **CVE-2026-52878 / GHSA-w4c6-7r69-w7j9** (Jun 5, 2026, High): klever-go REST API slow-header connection exhaustion via Gin `Engine.Run()` — **Gin-specific DoS pattern**. Fix: use `http.Server` directly with `ReadHeaderTimeout`/`ReadTimeout`/`WriteTimeout`/`IdleTimeout` (see `security.md`).
- **CVE-2026-42507** (Jun 4, 2026, Medium): Go `net/textproto` log injection — affects all Go HTTP servers including Gin. Patched in upcoming Go 1.25.12 / 1.26.5.
- **CVE-2026-42500** (May 29, 2026, Medium): Go `image` BMP paletted-image panic. Affects image-upload handlers. Fixed in `golang.org/x/image` v0.41.0+.
- **CVE-2026-39822** (embargoed): Go runtime security fix landing in Go 1.25.12 / 1.26.5.
- **CVE-2026-22786** (Jan 12, 2026, High): gin-vue-admin path traversal in file upload — generally applicable lesson for Gin handlers (sanitize filenames with `filepath.Base` + regex whitelist).

### Go 1.27 Development Status
### ⚠️ CRITICAL: Always Verify Versions Against go.dev/dl

The Go release cycle does NOT always produce security patch releases on the dates previous research suggested. **Before citing any specific patch version (e.g., go1.26.5, go1.25.12), always verify against the official source:**

```bash
curl -s https://go.dev/dl/?mode=json | python3 -c "import sys,json; [print(p['version'], p.get('stable','')) for p in json.load(sys.stdin)[:10]]"
```

Previous research incorrectly stated go1.26.5 and go1.25.12 existed as security patches. **The authoritative go.dev/dl API (verified 2026-06-21 12:04 UTC) confirms the latest stable versions are still go1.26.4 and go1.25.11** — but the dev.golang.org/release dashboard shows both patches are **imminent** (7 CLs for 1.26.5, 3 CLs for 1.25.12, as of 2026-06-17). **Always re-verify at go.dev/dl before deploying — patches may have shipped.**

### Go 1.27 Development Status
- Go 1.27 release freeze began **May 20, 2026** — **31 days in as of June 20, 2026**, RC1 tagged June 18, 2026 (see top of section)
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

### Verified Versions (2026-06-22 12:04 UTC — go.dev/dl API)

- **Gin v1.12.0** — released 2026-02-28, current latest (GitHub API confirmed)
- **Gin v1.13** — milestone #28, due 2026-06-30, **~65.7% complete (23/35 closed, 12 open)**, not yet released (verified 2026-06-24 00:06 UTC via GitHub API; was 22/35 = ~62.9% on 2026-06-23 12:13 UTC, was 21/33 = ~63.6% on 2026-06-23 00:11 UTC, was 17/30 = ~56.7% on 2026-06-22 12:04 UTC; total grew 33 → 35 as PR #4708 was added to the milestone)
- **Go 1.26.4** — **current stable** (verified via go.dev/dl; 2026-06-22 12:04 UTC)
- **Go 1.25.11** — **previous stable** (verified via go.dev/dl; 2026-06-22 12:04 UTC)
- **Go 1.27** — **RC1 released 2026-06-18** (verified via go.dev/dl/?mode=json 2026-06-25 00:15 UTC), final expected August 2026
- **go-redis v9.21.0** — latest stable (2026-06-22, supersedes v9.20.1)
- **golang.org/x/image v0.43.0+** — required to avoid BOTH CVE-2026-42500 BMP decode panic AND CVE-2026-46602 TIFF tile-size unbounded-memory DoS (new CVE published 2026-06-25)
- **GORM v1.31.2** — latest stable (2026-06-22, supersedes v1.31.1)
- **jackc/pgx v5 v5.10.0** — latest stable (Jun 3, 2026); **CVE-2026-33816 fixed in v5.9.0+** (critical memory-safety, CVSS 9.8); CVE-2026-33815 also patched in v5.10.0
- **golang-jwt/jwt v5.3.1** — latest stable
- **go-jose/go-jose** — if used (OIDC, JWE workflows): require **v3.0.5+** or **v4.1.4+** for **CVE-2026-34986 / GHSA-78h2-9frx-2jm8** (DoS panic in JWE decryption when `alg` is a `KW` key-wrap algorithm and `encrypted_key` is empty; CVSS 7.5; disclosed 2026-03-31)
- **golang.org/x/crypto v0.53.0** — latest stable
- **golang.org/x/net v0.55.0** — latest stable
- **golang.org/x/sys v0.46.0** — comes with Go 1.26 toolchain
- **golang.org/x/text v0.38.0** — latest stable
- **golang.org/x/arch v0.28.0** — latest stable
- **golang.org/x/sync v0.21.0** — errgroup, semaphore — used in concurrency.md patterns
- **quic-go v0.60.0** — HTTP/3 support (Gin v1.12+); Go 1.25+ required; FIPS 140-3 ready (Go 1.26+)
- **gin-contrib/cors v1.7.7** — CORS middleware
- **golang-migrate/migrate v4.19.1** — SQL migrations
- **pressly/goose v3.27.1** — SQL + Go migrations (Apr 24, 2026); min Go 1.25; Provider API with `WithLogger` for `slog`
- **ariga/atlas v1.2.0** — declarative schema-as-code + versioned auto-planning (Apr 10, 2026); reads `golang-migrate`/`goose`/`dbmate`/`flyway`/`liquibase` directory formats
- **squaredup/squawk (latest)** — PostgreSQL migration linter; CI guard for dangerous DDL

### Sources
- https://go.dev/dl/?mode=json (authoritative — verified 2026-06-22 12:04 UTC)
- https://github.com/gin-gonic/gin/releases (authoritative — verified 2026-06-22 12:04 UTC)
- https://github.com/gin-gonic/gin/milestones (Gin v1.13 milestone progress, verified 2026-06-22 12:04 UTC)
- https://go.dev/doc/go1.26 (Go 1.26 release notes — breaking changes verified)
- https://go.dev/doc/go1.27 (Go 1.27 release notes)
- https://github.com/golang/go/issues/76474 (Go 1.27 tracking)
- https://dev.golang.org/release (Go release dashboard)
- https://github.com/gin-gonic/gin/milestone/28 (Gin v1.13 milestone)
- https://github.com/golang/go/issues/75223 (Go 1.26 `net/url` host colon rejection)
- https://github.com/go-jose/go-jose/security/advisories/GHSA-78h2-9frx-2jm8 (go-jose CVE-2026-34986, JWE DoS)


---

## Updated from Research (2026-06-22, 12:04 UTC)

### Go Release Dashboard Snapshot Change

- **Go 1.26.5**: CL count dropped from 7 → 6 since the 2026-06-22 00:04 snapshot. The `crypto/tls` FIPS+`InsecureSkipVerify` CL that was added on 2026-06-21 is **no longer listed** on the release dashboard as of 2026-06-22 12:04 UTC. Likely reverted for further work; not in this patch cycle.
- **Go 1.25.12**: CL count dropped from 4 → 3 since the 2026-06-22 00:04 snapshot. Same `crypto/tls` FIPS+`InsecureSkipVerify` CL was the one removed. Effect: FIPS 140-3 + `InsecureSkipVerify` fix is now expected in a **later** patch release (1.25.13 / 1.26.6), not 1.25.12 / 1.26.5.
- The remaining CLs in both releases are unchanged (same `cmd/compile` slicemask, `cmd/fix` misleading message, runtime version parsing, runtime RWMutex ppc64le, security CVE-2026-39822).
- **Action for agents:** do not rely on the FIPS+`InsecureSkipVerify` fix landing in the upcoming 1.25.12 / 1.26.5 release. Pin to a later release if your FIPS deployment depends on it.

### Go 1.27 Release Status (RC1 Phase)
- Release freeze started **May 20, 2026** → **33 days in as of June 22, 2026** (was 32 on 2026-06-21). No RC1 tagged yet.

### Gin v1.13 Milestone Progress
- 21/33 issues closed (~63.6%), 12 open. Was 17/30 (~56.7%) on 2026-06-22 12:04 UTC, was 15/27 (~55.6%) on 2026-06-21. The total grew 30 → 33 (3 new issues added to the milestone) and the closed count grew 17 → 21 (4 issues closed net) in the past 12 hours. Four new PRs merged since the 2026-06-22 18:04 UTC snapshot:
  - **#4713** `chore(deps): bump github.com/quic-go/quic-go to v0.60.0` (merged) — bumps Gin v1.13's HTTP/3 dependency
  - **#4702** `fix(context): skip chmod on pre-existing dirs in SaveUploadedFile` (merged) — prevents unnecessary `chmod` calls when uploading files into pre-existing directories (avoiding noisy `EPERM` errors in shared-hosting scenarios)
  - **#4709** `test(context): use t.TempDir() for SaveUploadedFile permission test on WSL` (merged) — test infra fix
  - **#4699** `test(response_writer): add tests for Flush() with and without http.Flusher` (merged) — covers the new Flush() behavior on `ResponseWriter`
  - **#4698** `fix(recovery): record recovered panics in c.Errors` (merged) — panics caught by Recovery middleware are now logged into `c.Errors` so they show up via `c.Errors.ByType(ErrorTypePanic)`; previously only stdout/stderr
  - **#4695** `fix(context): Copy() copies Errors and Accepted fields` (merged) — fixes a long-standing bug where `c.Copy()` (used for goroutine handoff) dropped `c.Errors` and `c.Accepted`, causing silent data loss
- Notable in-flight PRs (verified via GitHub):
  - **#4543** `feat(binding): add support for binding whole request at once` (open, in progress) — would let handlers bind all sources (JSON body, query, headers, URI params) into a single struct with one `ShouldBind` call.
  - **#4499** `Changed trailing slash redirection behaviour` (open, in progress) — breaking change to Gin's redirect handling; Gin's current default 301-redirects `/foo/` → `/foo` (or vice-versa) for routes registered with the opposite trailing slash. The PR changes this default. Worth reviewing for any service that relies on the current redirect behavior.
  - **#4498** `fix(form): correctly differentiate between nil / present-but-empty slices` (open, in progress) — fixes a long-standing ambiguity in form binding where `?tags=` (present but empty) was indistinguishable from `?tags` (missing). After the fix, `tags=[]` vs `nil` will be distinguishable. May affect existing tests.
  - **#4506** `chore(response_writer): add Unwrap() method to ResponseWriter interface` (open, in progress) — needed for `errors.As` to traverse the writer chain (e.g. for `errors.Is(err, http.ErrHandlerTimeout)`). Low-impact but enables better error inspection.
  - **#4599** `refactor(gin): migrate IP handling to net/netip package` (open, in progress) — replaces internal `net.ParseIP` usage with `netip.ParseAddr`. **Breaking for downstream code** that relies on `c.ClientIP()` returning a `net.IP` — it will become `netip.Addr`. Watch for migration notes when v1.13 ships.
  - **#4674** `fix(tree): use url.PathUnescape for path parameters` (open, in progress) — security/correctness fix; replaces custom unescape logic with stdlib `url.PathUnescape`, eliminating a class of path-traversal-adjacent parsing bugs (relevant to any service that uses path params with `%xx` escapes, e.g. file IDs, slug routing, OAuth callback state)
  - **#4662** `feat: add InitSSE(), SSEStream() and fix deprecated CloseNotifier` (open, in progress) — adds proper Server-Sent Events helpers to Gin, including a one-shot `SSEStream()` for streaming a sequence of events. Removes the deprecated `CloseNotifier()` calls in favor of `Request.Context().Done()`. Worth adopting if you build any LLM-token-stream or event-stream endpoints on Gin.
  - **#4569** `feat(engine): add H2CConfig to allow configuring h2c http2.Server` (open, in progress) — exposes H2C (HTTP/2 cleartext) server configuration via `engine.H2CConfig`. Previously H2C was hardcoded with default settings; now you can set timeouts, read/write deadlines, and MaxConcurrentStreams for in-cluster H2C traffic.
  - **#4447** `avoid double unescaping` (open, in progress) — paired with #4674 above; prevents URL params from being unescaped twice (once by the router, once by the handler) which was a source of subtle bugs.
  - **#4217** `Replace deprecated CloseNotify with Request.Context().Done()` (open, in progress) — companion to #4662; finishes the migration off `http.CloseNotifier` (deprecated since Go 1.16).

### Go 1.26 Breaking Changes Now Documented

Added a new "Go 1.26 Breaking Changes Affecting Gin Handlers" section to versions.md covering:
- `net/url.Parse` host colon rejection (OAuth/OIDC redirect validation, webhook URL parsing)
- Default-on post-quantum TLS key exchanges (`SecP256r1MLKEM768`, `SecP384r1MLKEM1024`)
- Green Tea GC default-on (benchmark delta; positive)
- `go test` vet now runs `stringer` check (CI impact)
- `url.ParseRequestURI` behavior change for malformed hosts

### Verified Versions (Snapshot 2026-06-22 12:04 UTC)

All versions still current — no new releases since 2026-06-22 00:04 UTC snapshot:
- Go 1.26.4 (current stable), Go 1.25.11 (previous stable), Go 1.27 (release freeze, no RC1)
- Gin v1.12.0 (latest), Gin v1.13 (milestone progress updated)
- quic-go v0.60.0, go-redis v9.21.0, GORM v1.31.2, jackc/pgx v5.10.0, golang-jwt/jwt v5.3.1
- goose v3.27.1, atlas v1.2.0, gin-contrib/cors v1.7.7, golang-migrate v4.19.1

### Sources for This Update
- https://dev.golang.org/release (re-verified 2026-06-22 12:04 UTC)
- https://github.com/gin-gonic/gin/milestone/28 (Gin v1.13 — re-verified 2026-06-22 12:04 UTC)
- https://github.com/gin-gonic/gin/pulls?q=is%3Apr+milestone%3Av1.13 (open PRs in v1.13)
- https://go.dev/doc/go1.26 (Go 1.26 release notes — breaking changes section)
- https://github.com/golang/go/issues/75223 (url.Parse host colon rejection tracking)

---

## Updated from Research (2026-06-22, 18:04 UTC)

### May 22, 2026 golang.org/x/crypto + x/net + x/sys security batch — now covered in security.md

The skill previously documented the May 22 CVE batch only as bare CVE numbers in the dependency table (x/crypto v0.53.0, x/net v0.55.0). Detailed impact analysis for each CVE is now in `security.md` (9 new entries added).

**Quick reference — minimum versions required for May 22 CVE batch:**

| Package | Min Version | CVEs Fixed |
|---|---|---|
| `golang.org/x/net` | **v0.55.0** | CVE-2026-39821 (CRITICAL idna), CVE-2026-25680 (HTML CPU DoS), CVE-2026-42502 (HTML Render XSS) |
| `golang.org/x/crypto` | **v0.53.0** | CVE-2026-46598 (ssh/agent ed25519 panic), CVE-2026-46597 (ssh AES-GCM panic), CVE-2026-46595 (ssh callback authz bypass), CVE-2026-42508 (CA SignatureKey revocation), CVE-2026-39828/835/827/830/831/829 (ssh hardening batch — 6 CVEs) |
| `golang.org/x/sys` | **v0.46.0** | CVE-2026-39824 (Windows NTUnicodeString overflow — already in Go 1.26 toolchain) |

### Verification re-run (2026-06-22 18:04 UTC)
- **Go 1.26.4** — still current stable (no new release in past 6 hours)
- **Go 1.25.11** — still previous stable
- **Go 1.27** — release freeze now **35 days in** (was 34 on 2026-06-23 12:13 UTC; freeze started May 20, 2026)
- **Gin v1.12.0** — still latest
- **Gin v1.13** — milestone progress: still tracking ~17/30 (~56.7%) from the 12:04 UTC snapshot; re-verified on GitHub, no new merged PRs in past 6 hours
- All dependency versions unchanged (no releases in past 6 hours)

### Action for agents deploying new Gin services today
1. Pin `golang.org/x/net >= v0.55.0` in `go.mod` (clears idna critical CVE)
2. Pin `golang.org/x/crypto >= v0.53.0` (clears 9 ssh CVEs)
3. Pin `golang.org/x/sys >= v0.46.0` (clears Windows NTUnicodeString; free with Go 1.26)
4. Add `govulncheck ./...` to CI to catch transitive dependencies missing these floors
5. If using `html.Render` for user content, switch to `bluemonday` immediately (CVE-2026-42502)

### Sources for this update
- https://groups.google.com/g/golang-announce (re-verified 2026-06-22 18:04 UTC)
  - x/crypto batch: https://groups.google.com/g/golang-announce/c/a082jnz-LvI
  - x/net batch: https://groups.google.com/g/golang-announce/c/iI-mYSI0lu8
  - x/sys batch: https://groups.google.com/g/golang-announce/c/6MMI8Lj-Atg
- https://pkg.go.dev/vuln/list (GO-2026-5013 through GO-2026-5033)
- https://go.dev/dl/?mode=json (re-verified 2026-06-22 18:04 UTC — Go 1.26.4 still current)
- https://github.com/gin-gonic/gin/milestone/28 (Gin v1.13 milestone — re-verified 2026-06-22 18:04 UTC)


---

## Updated from Research (2026-06-23, 00:11 UTC)

### Go Release Dashboard Snapshot Change

- **Go 1.26.5**: CL count grew from 6 → **7** since the 2026-06-22 18:04 UTC snapshot. A new `cmd/compile: internal compiler error invalid heap allocated var without Heapaddr [1.26 backport]` CL has appeared on the dashboard. The `crypto/tls` FIPS+`InsecureSkipVerify` CL remains pulled.
- **Go 1.25.12**: CL count grew from 3 → **4** since the 2026-06-22 18:04 UTC snapshot. Same `cmd/compile: internal compiler error invalid heap allocated var without Heapaddr [1.25 backport]` CL is the new addition. FIPS CL remains pulled.
- The new `cmd/compile` CL is an ICE (internal compiler error) regression fix — **not** a security patch, but it does affect `cmd/compile` builds that hit the `invalid heap allocated var without Heapaddr` error path. If your CI is currently failing on a small percentage of Go 1.25/1.26 builds with this error, the upcoming patch releases will resolve it.

### Go 1.27 Release Status (RC1 Phase)
- Release freeze started **May 20, 2026** → **36 days in as of June 25, 2026** (was 35 on 2026-06-24, 34 on 2026-06-23, 33 on 2026-06-22, 32 on 2026-06-21). **RC1 tagged 2026-06-18** (supersedes the freeze-day counter for release scheduling — expect RC2 ~3 weeks out, final ~August 2026).

### Gin v1.13 Milestone Progress
- **21/33 issues closed (~63.6%)**, 12 open. Was 17/30 (~56.7%) on 2026-06-22 12:04 UTC. Six new merged PRs since the last update (4 in the past 12 hours alone): #4713 (quic-go v0.60.0), #4709, #4702, #4699, #4698, #4695.
- New in-flight PRs of note (added to v1.13 milestone since last update):
  - **#4674** `fix(tree): use url.PathUnescape for path parameters` — security-relevant path-param escape fix
  - **#4662** `feat: add InitSSE(), SSEStream()` — proper SSE helper API
  - **#4569** `feat(engine): add H2CConfig` — H2C server configuration
  - **#4447** `avoid double unescaping` — paired with #4674
  - **#4217** `Replace deprecated CloseNotify with Request.Context().Done()` — finishes CloseNotifier migration
- v1.13 is on pace to ship by the June 30, 2026 milestone date (~7 days away).

### Verification re-run (2026-06-23 00:11 UTC)
- **Go 1.26.4** — still current stable (no new release in past 6 hours)
- **Go 1.25.11** — still previous stable
- **Gin v1.12.0** — still latest released
- **Gin v1.13** — milestone now at 21/33 (~63.6%) — see above
- All dependency versions unchanged (no releases in past 6 hours)

### Action for agents
1. No urgent action items this cycle. The previous update's CVE-floor requirements still hold (x/net ≥ v0.55.0, x/crypto ≥ v0.53.0, x/sys ≥ v0.46.0).
2. When Gin v1.13 ships (~7 days), review the migration notes — `c.ClientIP()` will change type to `netip.Addr` (PR #4599), and trailing-slash redirect behavior will change (PR #4499).
3. If you're building SSE endpoints (e.g. for LLM token streaming), PR #4662 will add proper helpers — wait for v1.13 or backport the pattern.
4. If you terminate H2C traffic in-cluster, PR #4569 will let you configure `http2.Server` timeouts; useful for backpressure.

### Sources for this update
- https://dev.golang.org/release (re-verified 2026-06-23 00:11 UTC — 1.26.5: 7 CLs, 1.25.12: 4 CLs)
- https://github.com/gin-gonic/gin/milestone/28 (Gin v1.13 — re-verified 2026-06-23 00:11 UTC: 21/33 closed)
- https://github.com/gin-gonic/gin/pulls?q=is%3Apr+milestone%3Av1.13 (open PRs in v1.13)
- https://go.dev/dl/?mode=json (re-verified 2026-06-23 00:11 UTC — Go 1.26.4 still current)
- https://api.github.com/repos/gin-gonic/gin/commits?per_page=5 (most recent merged PRs)


---

## Updated from Research (2026-06-23, 06:16 UTC)

### Six-Hour Re-Verification (No Material Change to Dashboards)

This is a re-verification cycle run 6 hours after the 2026-06-23 00:11 UTC snapshot. The Go release dashboard, Go 1.27 freeze state, and Gin v1.13 milestone progress are all **unchanged** from the prior snapshot. The substantive finding this cycle is two notable in-flight PRs in gin-gonic/gin that were not captured in the 00:11 UTC update.

### No-change confirmations

- **Go release dashboard** (re-verified 2026-06-23 06:11 UTC): Go 1.26.5 still at **7 CLs**, Go 1.25.12 still at **4 CLs**. The `cmd/compile: internal compiler error invalid heap allocated var without Heapaddr` CL (added 00:11 UTC snapshot) is still present. The `crypto/tls` FIPS+`InsecureSkipVerify` CL remains pulled from both release branches.
- **Go 1.27 release freeze**: still **35 days in** as of June 24, 2026 (started May 20, 2026). No RC1 tagged.
- **Gin v1.13 milestone**: still **21/33 closed (~63.6%)**, 12 open. No new merged PRs in the past 6 hours.
- **Recently merged Gin PRs** (re-fetched via `api.github.com/repos/gin-gonic/gin/pulls?state=closed&per_page=10`): identical to 00:11 UTC snapshot — top 6 merged PRs are still #4713, #4709, #4702, #4699, #4698, #4695.
- **Go stable releases**: Go 1.26.4 still current stable, Go 1.25.11 still previous stable, no new release shipped in the past 6 hours.
- **All dependency versions**: unchanged (no new releases on go-redis, GORM, pgx, golang-jwt, goose, atlas, quic-go, gin-contrib in the past 6 hours).

### New in-flight PRs (NOT in 00:11 UTC snapshot — added 2026-06-23 06:16 UTC)

Two PRs opened before the 00:11 UTC snapshot but missed by that update's PR sweep. Both are worth flagging now because v1.13 ships in ~7 days:

- **#4714** `fix: flush status immediately for no-body response codes` (opened 2026-06-22, NOT YET in v1.13 milestone) — Correctness fix: `ctx.Status(code)` does not flush the status code to the underlying `http.ResponseWriter` when no body is written (statuses 1xx, 204, 304). `httptest.ResponseRecorder` sees 200 instead of 204 for `c.Status(http.StatusNoContent)`. Root cause is that `c.Writer.WriteHeader(code)` only buffers the status; `WriteHeaderNow()` is only triggered by a subsequent `Write()`. Fix mirrors `ctx.Render()`'s existing behavior — flush immediately when `bodyAllowedForStatus()` returns false. Tests cover 100/101/102/204/304 (flush) and 200 (buffer until Write). Closes #4071. **Impact**: anyone using `httptest.NewRecorder()` for unit tests of no-body endpoints, or any reverse proxy / middleware that inspects status before body, will get correct behavior on v1.13 (or sooner if backported).
- **#4712** `feat(render)!: make msgpack/bson/yaml/toml/protobuf opt-in subpackages` (opened 2026-06-21, NOT YET in v1.13 milestone, labeled `break-backward, refactor`) — **BREAKING CHANGE** PR that moves MessagePack, BSON, YAML, TOML, and ProtoBuf out of Gin's core into opt-in subpackages at `github.com/gin-gonic/gin/render/<format>`. Root cause: `binding.Default()` (reachable from any `ShouldBind` call) switched over every concrete binding, and the package-level `var MsgPack/BSON/... = xBinding{}` kept each as a link root the Go linker could not dead-code-eliminate, bloating every binary. Measured impact: a minimal JSON-only Gin binary drops from **~13 MB stripped to ~6.5 MB stripped** (~50% reduction) just by not linking ugorji msgpack (~6 MB) and mongo-driver/bson (~0.25 MB+). Migration pattern: import the subpackage for its `init()` side effect, then use the helper:
  ```go
  import _ "github.com/gin-gonic/gin/render/msgpack"
  // ...
  msgpack.Render(c, http.StatusOK, obj) // replaces c.MsgPack
  ```
  **Impact**: this is the single largest binary-size win proposed for v1.13, but downstream callers of `c.MsgPack`, `c.YAML`, `c.TOML`, `c.ProtoBuf`, `c.BSON`, or their binding equivalents will not compile until they adopt the subpackage imports. Watch this PR closely — if it lands in v1.13, a `find . -name '*.go' | xargs grep -lE '\.MsgPack|\.YAML|\.TOML|\.ProtoBuf|\.BSON'` sweep across your services will be required before upgrading.

### Action for agents

1. **No urgent action items** — the CVE-floor requirements from 2026-06-22 18:04 UTC still hold (x/net ≥ v0.55.0, x/crypto ≥ v0.53.0, x/sys ≥ v0.46.0). The pending Go 1.26.5 / 1.25.12 patches are still imminent but not yet shipped.
2. **Before upgrading to Gin v1.13** (~7 days away): audit for `c.MsgPack`, `c.YAML`, `c.TOML`, `c.ProtoBuf`, `c.BSON` callsites if #4712 lands — they will need to switch to the `render/<format>` subpackage helpers.
3. **Test correctness with no-body statuses**: if your test suite uses `httptest.NewRecorder()` and asserts on `recorder.Code` after `c.Status(http.StatusNoContent)` etc., the test currently passes only by accident — `recorder.Code` will be 200 today and 204 after #4714 lands. Update tests to expect the correct code either way.
4. **Re-verify Go release dashboard** before the next deploy: `curl -s https://dev.golang.org/release | grep -E 'Go1\.(26\.5|25\.12)'` — if a new CL appears, run the impact analysis in this skill.

### Sources for this update
- https://dev.golang.org/release (re-verified 2026-06-23 06:11 UTC — 1.26.5: 7 CLs, 1.25.12: 4 CLs; no change from 00:11 UTC snapshot)
- https://github.com/gin-gonic/gin/milestone/28 (Gin v1.13 — re-verified 2026-06-23 06:16 UTC: 21/33 closed, no change)
- https://api.github.com/repos/gin-gonic/gin/pulls?state=closed&per_page=10 (re-verified 2026-06-23 06:16 UTC — same top 6 merged PRs as 00:11 UTC snapshot)
- https://api.github.com/repos/gin-gonic/gin/issues/4714 (status flush fix — full body reviewed)
- https://api.github.com/repos/gin-gonic/gin/issues/4712 (codec opt-in subpackages — full body reviewed, includes binary-size measurements)
- https://go.dev/dl/?mode=json (re-verified 2026-06-23 06:16 UTC — Go 1.26.4 still current, no new release)


---

## Updated from Research (2026-06-23, 12:13 UTC)

### Six-Hour Re-Verification — Gin v1.13 validator Security Bump + Floor-Piercing Risk

This is the third 6-hour cycle in 24 hours. The Go release dashboard is unchanged (1.26.5: 7 CLs, 1.25.12: 4 CLs). The Go 1.27 freeze is still day 34. The substantive finding is **PR #4707 merged 5 minutes before this cron ran** — a security-driven validator bump that creates a new floor-piercing risk for downstream users adopting Gin v1.13.

### Event 1 — PR #4707 MERGED into gin-gonic/gin master

- **PR #4707** `update validator library to version 10.30.3` — **merged 2026-06-23T12:08:30Z** (~5 minutes before this verification cycle)
- Bumps `github.com/go-playground/validator/v10` from **v10.30.1 → v10.30.3** in `gin-gonic/gin` master go.mod (PR skipped v10.30.2 entirely).
- **PR body explicitly states**: *"Bump github.com/go-playground/validator/v10 from v10.30.1 to v10.30.3 — This fixes multiple critical vulnerabilities from golang.org/x/crypto"*. Labeled `dependencies`.
- **Gin master go.mod changes** (6 additions, 6 deletions):
  - `github.com/go-playground/validator/v10 v10.30.1` → `v10.30.3` (direct dep)
  - `golang.org/x/crypto v0.49.0` → `v0.52.0` (indirect, via validator)
  - `golang.org/x/text v0.35.0` → `v0.37.0` (indirect, via validator)
  - `golang.org/x/sys v0.42.0` → `v0.45.0` (indirect, via validator)
- **Validator v10.30.3 release notes (2026-05-29)**: includes dependabot bumps `golang.org/x/crypto v0.49.0→v0.50.0→v0.51.0→v0.52.0` and `golang.org/x/text v0.35.0→v0.36.0→v0.37.0` across PRs #1558/#1559/#1571/#1572/#1580 — plus 13 feature commits (UUID case-insensitive, NoneOf validator, bcp47_strict_language_tag, RFC 952 hostname trailing-hyphen fix, cron regex anchor fix, `origin` URL validator, dead-code-elimination build-size reduction, timezone translation support, MIME-type refactor).
- PR was opened 2026-06-15 (8 days before merge). Now in v1.13 milestone.

### Event 2 — Gin v1.13 milestone moved 21/33 → 22/35 (~62.9%)

- **22/35 issues closed**, 13 open. Was **21/33 (~63.6%)** at 06:16 UTC snapshot 6 hours ago.
- Numerator: +1 (PR #4707 closed). Denominator: +2 (two new issues added to milestone — likely the in-flight #4712 codec refactor or another tracked issue).
- Net effect: **progress dropped from ~63.6% to ~62.9%** because the milestone scope expanded faster than the close rate.
- v1.13 still on pace to ship by **June 30, 2026** (~7 days away).

### 🚨 Floor-Piercing Risk (CRITICAL — affects ALL downstream Gin v1.13 adopters)

**The Problem**: The skill's documented CVE floor (set in 2026-06-22 18:04 UTC update) requires:

- `golang.org/x/crypto ≥ v0.53.0` (clears 9 ssh CVEs from May 22 batch)
- `golang.org/x/sys ≥ v0.46.0` (clears CVE-2026-39824 Windows NTUnicodeString overflow)
- `golang.org/x/net ≥ v0.55.0` (clears CVE-2026-39821 critical idna CVSS 10.0)

**What Gin v1.13 will ship with** (after PR #4707 merges into v1.13):

- **x/crypto**: floor v0.53.0+ → Gin v1.13 will ship v0.52.0 (indirect via validator v10.30.3) → **BELOW floor by 1 minor**
- **x/sys**: floor v0.46.0+ → Gin v1.13 will ship v0.45.0 (indirect via validator v10.30.3) → **BELOW floor by 1 minor**
- **x/net**: floor v0.55.0+ → Gin v1.13 will ship v0.55.0 (direct) → **at floor** ✅
- **x/text**: latest v0.38.0 → Gin v1.13 will ship v0.37.0 (indirect via validator) → **1 minor below latest, no known CVE** ⚠️

**Impact**: Go's module Minimum Version Selection (MVS) picks the **highest** version required by any direct or transitive requirement. If a downstream service does NOT have an explicit `require golang.org/x/crypto v0.53.0` in its own go.mod (or higher), then after `go get github.com/gin-gonic/gin@v1.13.0 && go mod tidy`, the resolved `x/crypto` will be v0.52.0 — which is **below the CVE floor** and re-introduces the May 22 ssh CVE batch (CVE-2026-46598 ed25519 panic, CVE-2026-46597 AES-GCM panic, CVE-2026-46595 callback authz bypass, CVE-2026-42508 CA SignatureKey revocation, and 5 more ssh hardening CVEs).

**The remediation** is straightforward but **must be applied to every downstream service before upgrading to Gin v1.13**:

```go
// go.mod — REQUIRED for Gin v1.13 adoption
require (
    github.com/gin-gonic/gin v1.13.0
    golang.org/x/crypto v0.53.0  // pins above validator v10.30.3's v0.52.0
    golang.org/x/sys v0.46.0     // pins above validator v10.30.3's v0.45.0
)
```

Run `go mod tidy` after adding the pins. Verify with `go list -m golang.org/x/crypto golang.org/x/sys` — both should report the pinned version or higher.

**CI guard** (add to your pre-deploy pipeline):

```bash
#!/usr/bin/env bash
set -euo pipefail
MIN_CRYPTO="v0.53.0"
MIN_SYS="v0.46.0"
MIN_NET="v0.55.0"
fail=0
for mod in crypto sys net; do
    case $mod in
        crypto) min="$MIN_CRYPTO" ;;
        sys) min="$MIN_SYS" ;;
        net) min="$MIN_NET" ;;
    esac
    actual=$(go list -m "golang.org/x/$mod" | awk '{print $2}')
    if ! go list -m -f '{{if ge .Version "'"$min"'"}}{{.Version}}{{end}}' "golang.org/x/$mod" | grep -q .; then
        echo "❌ golang.org/x/$mod is $actual, below required $min"
        fail=1
    else
        echo "✅ golang.org/x/$mod $actual >= $min"
    fi
done
exit $fail
```

### No-change confirmations

- **Go release dashboard** (re-verified 2026-06-23 12:11 UTC): Go 1.26.5 still at **7 CLs**, Go 1.25.12 still at **4 CLs**. No new CLs in past 6 hours.
- **Go 1.27 release freeze**: still **35 days in** as of June 24, 2026. No RC1 tagged.
- **Go stable releases**: Go 1.26.4 still current stable, Go 1.25.11 still previous stable, no new release shipped in the past 6 hours.
- **Gin v1.13 in-flight PRs** (re-checked at 12:13 UTC): same as 06:16 UTC snapshot — #4714, #4712 still NOT in milestone; #4674, #4662, #4569, #4599, #4543, #4506, #4499, #4708 still open in milestone.
- **All other dependency versions**: unchanged (no new releases on go-redis, GORM, pgx, golang-jwt, goose, atlas, quic-go, gin-contrib in the past 6 hours).

### Action for agents

1. **🚨 CRITICAL — Floor pins before Gin v1.13 upgrade**: Add explicit `require golang.org/x/crypto v0.53.0` and `require golang.org/x/sys v0.46.0` to your go.mod **before** `go get github.com/gin-gonic/gin@v1.13.0`. Without these pins, `go mod tidy` will resolve x/crypto to v0.52.0 (below floor, re-introducing May 22 ssh CVE batch) and x/sys to v0.45.0 (below floor, re-introducing Windows NTUnicodeString overflow on Windows builds).
2. **CI guard**: add the bash script above (or equivalent) to your CI pipeline to catch regressions if a transitive dep ever bumps below floor.
3. **PR #4707 action items from 06:16 UTC snapshot still apply**: (a) audit for `c.MsgPack`/`c.YAML`/`c.TOML`/`c.ProtoBuf`/`c.BSON` callsites if PR #4712 lands; (b) update `httptest.NewRecorder()` assertions to expect correct status codes (200→204 for no-body codes) if PR #4714 lands.
4. **No Go release patch action yet** — Go 1.26.5 and 1.25.12 still on dashboard but not yet tagged/shipped. Re-verify `https://dev.golang.org/release` before the next deploy.
5. **Watch for validator v10.30.4+**: if dependabot bumps `x/crypto` to v0.53.0+ in validator itself, the floor-piercing risk resolves automatically. Watch https://github.com/go-playground/validator/releases for new tags.

### Sources for this update

- https://github.com/gin-gonic/gin/pulls/4707 (PR body, merged_at, labels — re-verified 2026-06-23 12:13 UTC)
- https://github.com/gin-gonic/gin/milestone/28 (v1.13 milestone — re-verified 2026-06-23 12:13 UTC: 22/35 closed)
- https://raw.githubusercontent.com/gin-gonic/gin/master/go.mod (master go.mod — re-verified 2026-06-23 12:13 UTC)
- https://raw.githubusercontent.com/go-playground/validator/v10.30.3/go.mod (validator v10.30.3 transitive requirements)
- https://raw.githubusercontent.com/go-playground/validator/v10.30.2/go.mod (validator v10.30.2 transitive requirements — for comparison)
- https://api.github.com/repos/go-playground/validator/releases/latest (v10.30.3 release notes — published 2026-05-29)
- https://dev.golang.org/release (re-verified 2026-06-23 12:11 UTC — 1.26.5: 7 CLs, 1.25.12: 4 CLs; no change)
- https://go.dev/dl/?mode=json (re-verified 2026-06-23 12:13 UTC — Go 1.26.4 still current, no new release)

## Auto-update 2026-06-24 00:06 UTC (Cycle)

Six-hour cron cycle. Re-verification + freeze day count bump + 1 missed-PR catch.

### Substantive findings

1. **Go 1.27 release freeze day count bumped**: **34 → 35 days in** as of June 24, 2026 (was 34 on 2026-06-23 12:13 UTC). Freeze started May 20, 2026. No RC1 tagged yet.
2. **Gin v1.13 milestone progress bumped**: **22/35 → 23/35 (~62.9% → ~65.7%)** — PR **#4708** "ci(golangci): replace interface{} with any (Go 1.18+)" **merged at 2026-06-23T12:14:04Z**, exactly **1 minute after** the prior skill update was committed at 12:13 UTC. Genuinely missed by the previous cycle. Trivial CI hygiene change — `golangci-lint` v1.64+ flags `interface{}`; cosmetic only. Does NOT affect the floor-piercing-risk callouts from PR #4707.
3. **Go release dashboard**: **unchanged** — Go 1.26.5 still **7 CLs** (#79875 slicemask, #79878 ppc64le RWMutex, #80098 heapaddr, #80103 runtime.Version, #80108 cmd/fix, #80114 slicemask detail, plus #CVE-2026-39822), Go 1.25.12 still **4 CLs** (#79875, #79878, #80098, #CVE-2026-39822).
4. **Go stable releases**: unchanged — Go 1.26.4 still current, Go 1.25.11 still previous. No patch shipped in the past 12 hours.
5. **Validator floor-piercing risk** (from prior cycle's PR #4707 finding): still active — `validator v10.30.3` transitively pins `x/crypto v0.52.0` and `x/sys v0.45.0`. CI guard script + explicit `require` lines in go.mod remain the recommended defense.
6. **No new CVEs** discovered in the past 12 hours affecting the Gin dependency tree.

### No-change confirmations

- **Gin v1.12.0** (released 2026-02-28) still the current stable. No v1.13 release tagged.
- **In-flight Gin PRs** still open in master: #4674 url.PathUnescape, #4662 SSE helpers, #4569 H2CConfig, #4599 net/netip, #4543 whole-request binding, #4506 ResponseWriter.Unwrap, #4499 trailing slash, #4498 form nil-vs-empty, plus #4712 / #4714 (still NOT yet in milestone as of this snapshot — same as 2026-06-23 06:16 UTC note).
- **go-redis, GORM, pgx, golang-jwt, goose, atlas, quic-go, gin-contrib**: all unchanged. No new releases in the past 12 hours.

### Action for agents

1. **No new floor-pin or migration action** — the `golang.org/x/crypto v0.53.0` / `golang.org/x/sys v0.46.0` go.mod pin requirement (from the prior cycle's PR #4707 finding) remains the only critical advisory.
2. **Go 1.26.5 / 1.25.12 still imminent** but not yet shipped. Re-verify `https://dev.golang.org/release` and `https://go.dev/dl/?mode=json` before the next deploy.
3. **Go 1.27 freeze day 35** — RC1 expected soon. Watch `https://go.dev/dl/?mode=json` for the RC tag.
4. **No new CVE or breaking change** to plan for in this 6-hour window.

### Sources for this update

- https://github.com/gin-gonic/gin/milestone/28 (v1.13 — re-verified 2026-06-24 00:06 UTC: **23/35 closed, ~65.7%**)
- https://github.com/gin-gonic/gin/pulls/4708 (PR #4708 merged 2026-06-23T12:14:04Z — 1 minute after prior cycle's commit; trivial CI lint change)
- https://api.github.com/repos/gin-gonic/gin/issues?milestone=28&state=closed (full milestone audit 2026-06-24 00:06 UTC)
- https://dev.golang.org/release (re-verified 2026-06-24 00:06 UTC — 1.26.5: 7 CLs, 1.25.12: 4 CLs; no change since 2026-06-23 12:11 UTC)
- https://go.dev/dl/?mode=json (re-verified 2026-06-24 00:06 UTC — Go 1.26.4 still current, no new release)
- https://api.github.com/repos/gin-gonic/gin/commits?per_page=10&since=2026-06-23T12:13:00Z (zero new master commits in past 12 hours — confirming no Gin releases happened)

## Auto-update 2026-06-24 12:09 UTC (Cycle)

Six-hour cron cycle. Pure re-verification — zero deltas across all tracked dashboards.

### Substantive findings

1. **Go 1.27 release freeze day count**: still **35 days in** as of June 24, 2026. Bumps to 36 on June 25, 2026. No RC1 tagged yet. (The freeze was May 20, 2026; June 24 = day 35 counting May 20 as day 0.)
2. **Gin v1.13 milestone progress**: still **23/35 closed (~65.7%)**. No new merged PRs in the past 12 hours. The closed-but-unmerged PR #4710 (chore(deps): bump the actions group) is a GitHub Actions dep bump and does not count toward the milestone (it was closed without merge, likely as a stale-supersede).
3. **Go release dashboard**: **unchanged** — Go 1.26.5 still **7 pending CLs** across `cmd/compile`, `cmd/fix`, `cmd/link`, `runtime`, `security`, `x/tools/go/analysis`; Go 1.25.12 still **4 pending CLs** across `cmd/compile`, `runtime`, `security`. Issue refs re-pulled at this snapshot: 1.26.5 = {`#77800`, `#79027`, `#79876`, `#79879`, `#79893`, `#80099`, `#80131`}; 1.25.12 = {`#79026`, `#79875`, `#79878`, `#80098`}.
4. **Go stable releases**: unchanged — Go 1.26.4 still current stable, Go 1.25.11 still previous stable, no patch shipped in the past 12 hours.
5. **Validator floor-piercing risk** (from prior cycle's PR #4707 finding): still active — `validator v10.30.3` transitively pins `x/crypto v0.52.0` and `x/sys v0.45.0`. No validator v10.30.4 yet.
6. **No new CVEs** discovered in the past 12 hours affecting the Gin dependency tree.

### No-change confirmations

- **Gin v1.12.0** (released 2026-02-28) still the current stable. No v1.13 release tagged.
- **In-flight Gin PRs** still open in master: #4674 url.PathUnescape, #4662 SSE helpers, #4569 H2CConfig, #4599 net/netip, #4543 whole-request binding, #4506 ResponseWriter.Unwrap, #4499 trailing slash, #4498 form nil-vs-empty, plus #4712 / #4714 (still NOT yet in milestone as of this snapshot — same status as 2026-06-23 06:16 UTC note).
- **go-redis, GORM, pgx, golang-jwt, goose, atlas, quic-go, gin-contrib**: all unchanged. No new releases in the past 12 hours.
- **Gin master branch commits**: zero new master commits touching the source tree in past 12 hours (only the closed-not-merged #4710 action bump). Confirmed via `https://api.github.com/repos/gin-gonic/gin/commits?per_page=20&since=2026-06-24T00:08:00Z`.

### Action for agents

1. **No new action** — all advisories from the prior cycles remain in force. The `golang.org/x/crypto v0.53.0` / `golang.org/x/sys v0.46.0` go.mod pin requirement (from the 2026-06-23 12:13 UTC PR #4707 finding) is still the only critical advisory.
2. **Go 1.26.5 / 1.25.12 still imminent** but not yet shipped. Re-verify `https://dev.golang.org/release` and `https://go.dev/dl/?mode=json` before the next deploy.
3. **Go 1.27 freeze day 35 → 36** transition: count remains 35 today (June 24). Watch `https://go.dev/dl/?mode=json` for the RC1 tag — typically drops 1–2 weeks after the freeze day-count crosses 40 in past Go cycles.
4. **No new CVE or breaking change** to plan for in this 6-hour window.

### Sources for this update

- https://github.com/gin-gonic/gin/milestone/28 (v1.13 — re-verified 2026-06-24 12:09 UTC: **23/35 closed, ~65.7%**; same as 2026-06-24 00:06 UTC)
- https://api.github.com/repos/gin-gonic/gin/pulls/4710 (PR #4710 — closed but NOT merged; GitHub Actions dep bump, stale-superseded)
- https://api.github.com/repos/gin-gonic/gin/commits?per_page=20&since=2026-06-24T00:08:00Z (zero new master source commits)
- https://dev.golang.org/release (re-verified 2026-06-24 12:09 UTC — 1.26.5: 7 CLs {`#77800`, `#79027`, `#79876`, `#79879`, `#79893`, `#80099`, `#80131`}, 1.25.12: 4 CLs {`#79026`, `#79875`, `#79878`, `#80098`}; counts unchanged since 2026-06-24 00:06 UTC)
- https://go.dev/dl/?mode=json (re-verified 2026-06-24 12:09 UTC — Go 1.26.4 still current stable, Go 1.25.11 still previous stable, no new release)


## Auto-update 2026-06-25 00:15 UTC (Cycle)

Six-hour cron cycle. **MAJOR FINDING: Go 1.27 RC1 shipped 2026-06-18** — the prior 8 cycles of "no RC1 yet" status are now obsolete. Substantive changes below; the top-of-file "## Go 1.27" section has been updated to reflect RC1 status.

### Substantive findings

1. **Go 1.27 RC1 RELEASED 2026-06-18** — `go1.27rc1` is now available in `https://go.dev/dl/?mode=json` (stable=False). Install: `go install golang.org/dl/go1.27rc1@latest && go1.27rc1 download`. Do not run in production; use to validate your test suite ahead of the August GA. The release-notes draft at [go.dev/doc/go1.27](https://go.dev/doc/go1.27) is now in `DRAFT RELEASE NOTES` state with confirmed section structure. The freeze-day counter is superseded by RC-phase tracking: RC1 (Jun 18) → expected RC2 ~3 weeks later → final ~August 2026.
2. **Release-freeze day count bumped to 36** (was 35 on 2026-06-24). Freeze started May 20, 2026; June 25 = day 36. No longer the leading metric for release readiness (RC phase is).
3. **Gin v1.13 milestone progress**: still **23/35 closed (~65.7%)**. No new merged PRs in the past 12 hours.
4. **Go release dashboard**: **unchanged** — Go 1.26.5 still **7 pending CLs**, Go 1.25.12 still **4 pending CLs**. Issue refs: 1.26.5 = {`#77800`, `#79027`, `#79876`, `#79879`, `#79893`, `#80099`, `#80131`}; 1.25.12 = {`#79026`, `#79875`, `#79878`, `#80098`}.
5. **Go stable releases**: unchanged — Go 1.26.4 current stable, Go 1.25.11 previous stable, no patch shipped.
6. **Validator floor-piercing risk** (from 2026-06-23 cycle's PR #4707 finding): still active — `validator v10.30.3` transitively pins `x/crypto v0.52.0` and `x/sys v0.45.0`. No validator v10.30.4 yet.
7. **No new CVEs** in the past 12 hours affecting the Gin dependency tree.

### New Go 1.27 RC1 features captured this cycle (not in prior skill versions)

Audit of the [Go 1.27 release notes](https://go.dev/doc/go1.27) revealed **11 features** that were missing from the prior skill's "New in Go 1.27" / "Other New Standard Library APIs" / "Tools Improvements" sections. Added in this cycle:

**Standard library additions:**
- `crypto/x509.SystemCertPool` now respects `SSL_CERT_FILE` / `SSL_CERT_DIR` on Windows and Darwin (security-relevant — affects any containerized Gin service that sets those env vars)
- `crypto/tls.ConnectionState.LocalCertificate` — chain presented TO the peer (pairs with existing `PeerCertificates`)
- `crypto/tls.QUICConfig.ClientHelloInfoConn` — for Gin services running HTTP/3 via custom `quic.Listener`
- `crypto/tls.Config.Rand` deprecated — use `testing/cryptotest.SetGlobalRandom` for deterministic TLS testing
- `crypto/ecdsa.PrivateKey.Sign` hash-length check — signs that previously silently produced broken signatures now error at sign time
- `database/sql.ConvertAssign` + `database/sql/driver.RowsColumnScanner` — for Gin projects that embed custom DB drivers
- `go/constant.StringLen` — for static-analysis tools built on top of Gin
- `go/token.File.String()` — small QoL for code-gen tools
- `compress/flate` speedup — **output bytes may differ** from Go 1.26 (same format, different implementation). **Downstream effect on `archive/zip`, `compress/gzip`, `compress/zlib`, `image/png` outputs** — Gin impact: golden-file tests for compressed assets will need a re-baseline.

**Tools / build additions:**
- `go doc package@version` syntax
- `go doc -ex` flag for executable examples
- `go test -json` `OutputType` field annotation
- `go tool trace -http=:PORT` now localhost-only by default (security)
- Linker `-macos` / `-macsdk` flags for macOS cross-compile
- `runtime/proof` goroutine labels in tracebacks (Go 1.27+ modules) — opt-out via `GODEBUG=tracebacklabels=0`

**BREAKING changes in Go 1.27 (must plan for):**
- **`asynctimerchan` GODEBUG setting removed permanently** — `time.Timer`/`time.Ticker` channels are **always unbuffered** now. Any Gin code that relied on the old buffered-1-channel semantics for non-blocking `select` patterns needs re-verification.
- **5 TLS GODEBUG settings removed permanently** — `tlsunsafeekm`, `tlsrsakex`, `tls3des`, `tls10server`, `x509keypairleaf`. Search your services + config for these strings before upgrading.
- **`bzr` version control support removed** — cannot fetch modules from bzr servers (already in skill).

### No-change confirmations

- **Gin v1.12.0** (released 2026-02-28) still current stable. No v1.13 release tagged.
- **Gin master** — last merged commit 2026-06-23 12:08 UTC (PR #4707 validator bump, already captured in 2026-06-23 12:13 UTC cycle). Zero new master source commits in the past 12 hours.
- **In-flight Gin PRs** still open: #4674, #4662, #4569, #4599, #4543, #4506, #4499, #4498, #4712, #4714 (statuses unchanged).
- **go-redis, GORM, pgx, golang-jwt, goose, atlas, quic-go, gin-contrib**: all unchanged.

### Action for agents

1. **RC1 validation is now actionable** — install `go1.27rc1`, run `go1.27rc1 test ./...` against your Gin services. File issues at [go.dev/issue](https://go.dev/issue) for any breakage. Pay special attention to: TLS configs that set removed GODEBUG flags, `time.Timer` select patterns, and compressed-asset byte-equality tests.
2. **`golang.org/x/crypto v0.53.0` / `golang.org/x/sys v0.46.0` go.mod pin** — still the only critical advisory (from 2026-06-23 12:13 UTC cycle's PR #4707 finding).
3. **Go 1.26.5 / 1.25.12 still imminent** but not yet shipped. Re-verify `https://dev.golang.org/release` before the next deploy.
4. **Gin v1.13** (~5 days away as of this cycle) — review migration notes for `c.ClientIP()` → `netip.Addr` (PR #4599), trailing-slash behavior (PR #4499), and `c.MsgPack`/`c.YAML`/`c.TOML`/`c.ProtoBuf`/`c.BSON` removal if PR #4712 lands.

### Sources for this update

- https://go.dev/dl/?mode=json (re-verified 2026-06-25 00:15 UTC — `go1.27rc1` present, Go 1.26.4 still current stable, Go 1.25.11 still previous stable)
- https://raw.githubusercontent.com/golang/go/release-branch.go1.27/VERSION (re-verified 2026-06-25 00:15 UTC — content `go1.27rc1\ntime 2026-06-18T17:05:58Z`, confirming RC1 tag date)
- https://raw.githubusercontent.com/golang/website/master/_content/doc/go1.27.md (Go 1.27 release notes source — full content audited 2026-06-25 00:15 UTC)
- https://github.com/golang/go/issues/78779 (Go 1.27 release notes tracking issue — open, no `okay-after-rc1` label yet)
- https://groups.google.com/g/golang-announce/c/Cu9HkstbtpA (RC1 announcement referenced by byteiota.com coverage 2026-06-21)
- https://github.com/gin-gonic/gin/milestone/28 (v1.13 — re-verified 2026-06-25 00:15 UTC: **23/35 closed, ~65.7%**; same as 2026-06-24 12:09 UTC)
- https://api.github.com/repos/gin-gonic/gin/commits?per_page=20&since=2026-06-24T12:09:00Z (zero new master source commits in past 12 hours)
- https://dev.golang.org/release (re-verified 2026-06-25 00:15 UTC — 1.26.5: 7 CLs, 1.25.12: 4 CLs; unchanged)
- https://byteiota.com/go-1-27-rc1-generic-methods-land-heres-what-changes-now (RC1 coverage 2026-06-21)
- https://nesbitt.io/2026/06/20/this-week-in-package-management.html (TWiPM mention of Go 1.27rc1)
- https://lmika.org/2026/06/19/go-rc-just-dropped-it.html (RC1 mention 2026-06-19)

## Auto-update 2026-06-25 06:09 UTC (Cycle)

Six-hour cron cycle. **Six-hour window yielded modest new activity**: 5 Gin issues/PRs updated, 1 PR (#4716) missed by the prior cycle. Go release dashboard unchanged at unique-CL level (Go 1.26.5 dashboard header counts "8" but only 7 unique issues — #77800 is counted under both `cmd/fix` and `x/tools/go/analysis` directories, same as in all prior cycles). Most important finding: PR #4660 documents a real, reproducible data race in `gin.Context.Set()` + `Context.Copy()` — security/correctness-relevant; flagged in `security.md` too.

### Substantive findings

1. **Gin v1.13 milestone progress**: still **23/35 closed (~65.7%)**. No new merged PRs in the past 6 hours.
2. **Go release dashboard**: **unchanged** at unique-issue level. Dashboard header counts (with double-counts):
   - Go 1.26.5 — header shows **8**, unique issues = **7** (counts #77800 twice: `cmd/fix` + `x/tools/go/analysis`): {#80099, #79876, #77800, #80131, #79879, #79893, #79027}. Issue list identical to 2026-06-25 00:15 UTC snapshot.
   - Go 1.25.12 — header shows **4**, unique issues = **4** (no double-counts): {#80098, #79875, #79878, #79026}. Unchanged.
3. **Go stable releases**: unchanged — Go 1.26.4 current stable, Go 1.25.11 previous stable, no patch shipped.
4. **Go 1.27 RC1 status**: unchanged — `go1.27rc1` (tagged 2026-06-18T17:05:58Z) still latest. No RC2 yet. RELEASE-NOTES draft at `https://go.dev/doc/go1.27` unchanged from prior cycle's audit.
5. **Validator floor-piercing risk** (from 2026-06-23 cycle's PR #4707 finding): still active — `validator v10.30.3` transitively pins `x/crypto v0.52.0` and `x/sys v0.45.0`. No validator v10.30.4 yet.
6. **No new CVEs** in the past 6 hours affecting the Gin dependency tree.

### New/updated Gin issues & PRs in the past 6 hours

Captured via `https://api.github.com/repos/gin-gonic/gin/issues?state=all&since=2026-06-25T00:15:00Z`:

| # | Type | Title | Status | Notes |
|---|------|-------|--------|-------|
| **#4660** | PR (open) | `fix(context): data race` | open since 2026-05-22, updated 2026-06-25 05:16 UTC | **CORRECTNESS/SECURITY-RELEVANT** — reproducible data race between `ctx.Set()` and `ctx.Copy()`. See below. |
| **#4701** | PR (open) | `feat(context): add AbortedByHandler() and AbortedBy() to track abort origin` | open since 2026-06-10, updated 2026-06-25 04:14 UTC | Observability feature for tracking which handler in the middleware chain called `Abort()`. Useful for logging/conditional middleware. Adds two new exported methods on `*gin.Context`. |
| **#4705** | PR (open) | `fix: encode non-BMP characters as UTF-16 surrogate pairs in AsciiJSON` | open since 2026-06-13, updated 2026-06-25 00:32 UTC | Bug fix in `AsciiJSON` rendering for emoji / CJK Extension B characters (>U+FFFF). Currently emits invalid 5+ hex-digit `\u1f389` instead of valid `\ud83c\udf89` surrogate pairs. Duplicate of #4693. |
| **#4693** | PR (open) | `fix(render): correctly escape non-BMP Unicode in AsciiJSON` | open since 2026-06-03, updated 2026-06-25 02:22 UTC | Same bug as #4705 — duplicate PR. Both authors independently identified the same `fmt.Appendf(buf, "\u%04x", r)` overflow bug. |
| **#4689** | PR (open) | `refactor(binding): simplify tryToSetValue option handling` | open since 2026-06-03, updated 2026-06-25 01:25 UTC | Cosmetic refactor of `tryToSetValue` in binding code. Removes redundant temporary variable and redundant empty-string check. The author themselves note "this issue is completely negligible." Low value; expect merge if maintainers feel like it. |

### Newly-captured PR missed by prior cycles: #4716

| # | Type | Title | Status | Notes |
|---|------|-------|--------|-------|
| **#4716** | PR (open) | `chore(deps): bump the actions group across 1 directory with 3 updates` | open since 2026-06-23 22:32 UTC | Dependabot batched bump: `actions/checkout` 6→7, `actions/cache` ?, `codecov/codecov-action` ?. Pure CI dep bump, NOT in v1.13 milestone, doesn't affect Go module consumers. The 2026-06-25 00:15 UTC cycle listed open PRs but omitted #4716 — added to the master open-PR list now. |

### New findings carried over from 2026-06-25 00:15 UTC cycle

Unchanged: PR #4707 (validator v10.30.3 floor-piercing risk → require explicit `golang.org/x/crypto v0.53.0` / `golang.org/x/sys v0.46.0` pins in go.mod for any Gin service adopting v1.13). Documented in `security.md`.

### In-flight Gin PRs — complete open list as of this snapshot

Per `https://api.github.com/repos/gin-gonic/gin/issues?state=all` filtered to open PRs in milestone/28 + active unmerged PRs:

- **In v1.13 milestone (open, not yet merged):** #4674 url.PathUnescape, #4662 SSE helpers, #4569 H2CConfig, #4599 net/netip, #4543 whole-request binding, #4506 ResponseWriter.Unwrap, #4499 trailing slash, #4498 form nil-vs-empty, #4483 multi-write warning, #4712 BREAKING render/<format> opt-in, #4714 ctx.Status no-body flush, #4696 rune-boundary safety
- **Open, NOT in v1.13 milestone:** #4701 AbortedByHandler (feature, observability), #4660 data race (correctness), #4705/#4693 AsciiJSON non-BMP (duplicate pair), #4689 tryToSetValue refactor (cosmetic), #4711 `go fix` application (chore), #4706 "my first contribution", #4703 OSS-Fuzz integration (large feature), #4694 codecov-action bump (chore), #4697 codecov/codecov-action 6→7 (chore, stale), #4716 actions group bump (chore)
- **Closed-not-merged:** #4710 (GitHub Actions bump — stale-superseded)

### Deep-dive: PR #4660 (data race)

**Severity: medium-high** for any Gin service that spawns goroutines that touch `*gin.Context`. Reproducer in the PR body:

```go
wg.Add(1)
go func() {
    defer wg.Done()
    for i := 0; i < n; i++ {
        ctx.Set("key", i)  // map write
    }
}()
wg.Add(1)
go func() {
    defer wg.Done()
    for i := 0; i < n; i++ {
        _ = ctx.Copy()     // map read (during Keys copy)
    }
}()
```

`go run -race` reports `WARNING: DATA RACE — Write at 0x... by goroutine 10`. Root cause: `Context.Copy()` performs a shallow copy of the `Keys map[any]any` reference without synchronization. The previous partial fix in PR #4695 (closed 2026-06-22) added `Errors` and `Accepted` to `Copy()` but missed `Keys` (the map itself, not the slice).

**Action for agents working on Gin services:**
1. **Audit** any code path where a goroutine spawned by a handler (or middleware) calls `ctx.Set()` / `ctx.Get()` / `ctx.Copy()` after the handler returns. Replace with a per-goroutine `context.Context` or serialize access with a mutex.
2. **Do not** depend on PR #4660 landing in Gin v1.13 — it is NOT in the v1.13 milestone. Pin a fork or apply the patch manually if your service is affected.
3. **Race-detector sweep recommended**: `go build -race` + load test on any Gin handler that fans out goroutines.

The fix from PR #4660 is small (sync.Mutex around the Keys map or deep-copy in Copy()). Codecov report attached to PR shows 98.37% coverage and tests added. Expect upstream merge in v1.13.x or v1.14 if maintainers re-scope the milestone.

### No-change confirmations

- **Gin v1.12.0** (released 2026-02-28) still current stable. No v1.13 release tagged.
- **Gin master** — last merged source commit 2026-06-23 12:08 UTC (PR #4707 validator bump). Zero new merged source commits in past 6 hours.
- **go-redis, GORM, pgx, golang-jwt, goose, atlas, quic-go, gin-contrib**: all unchanged. No new releases in past 6 hours.
- **Go 1.27 release-freeze day count**: 36 days in (May 20 freeze; June 25 = day 36). RC-phase tracking supersedes day-count.

### Action for agents

1. **Audit goroutine usage** in any Gin service for the data race pattern from PR #4660. This is the most important actionable item from this cycle.
2. **`golang.org/x/crypto v0.53.0` / `golang.org/x/sys v0.46.0` go.mod pin** still required for Gin v1.13 adopters (from PR #4707 finding in 2026-06-23 12:13 UTC cycle).
3. **Go 1.26.5 / 1.25.12 still imminent** but not yet shipped. Re-verify `https://dev.golang.org/release` before next deploy.
4. **PR #4701 (AbortedByHandler) and PR #4705/#4693 (AsciiJSON non-BMP)** — both are good-merge candidates; if you're running a Gin fork, consider cherry-picking once they land.
5. **No new CVE or breaking change** to plan for in this 6-hour window.

### Sources for this update

- https://go.dev/dl/?mode=json (re-verified 2026-06-25 06:09 UTC — `go1.27rc1` present, Go 1.26.4 current stable, Go 1.25.11 previous stable)
- https://raw.githubusercontent.com/golang/go/release-branch.go1.27/VERSION (re-verified 2026-06-25 06:09 UTC — `go1.27rc1`, time `2026-06-18T17:05:58Z`, unchanged)
- https://github.com/gin-gonic/gin/milestone/28 (v1.13 — re-verified 2026-06-25 06:09 UTC: **23/35 closed, ~65.7%**; same as 2026-06-25 00:15 UTC)
- https://api.github.com/repos/gin-gonic/gin/commits?per_page=20&since=2026-06-25T00:15:00Z (zero new master source commits in past 6 hours)
- https://dev.golang.org/release (re-verified 2026-06-25 06:09 UTC — 1.26.5 header=8 / unique=7; 1.25.12 header=4 / unique=4; same unique issue set as 2026-06-25 00:15 UTC)
- https://api.github.com/repos/gin-gonic/gin/issues?state=all&since=2026-06-25T00:15:00Z (5 issues/PRs updated in past 6 hours: #4660, #4701, #4705, #4693, #4689)
- https://api.github.com/repos/gin-gonic/gin/pulls/4660 (data race reproducer + fix — NOT in v1.13 milestone)
- https://api.github.com/repos/gin-gonic/gin/pulls/4716 (actions group bump, missed by prior cycle)
- https://api.github.com/repos/gin-gonic/gin/pulls/4701 (AbortedByHandler feature)
- https://api.github.com/repos/gin-gonic/gin/pulls/4705 (AsciiJSON non-BMP fix; duplicate of #4693)
- https://api.github.com/repos/gin-gonic/gin/pulls/4693 (AsciiJSON non-BMP fix; duplicate of #4705)
- https://api.github.com/repos/gin-gonic/gin/pulls/4689 (binding refactor)

## Auto-update 2026-06-25 12:07 UTC (Cycle)

Six-hour cron cycle. **Quiet 6-hour window**: zero new Gin master commits, zero new merged PRs, zero new CVEs, no Gin v1.13 milestone progress (still 23/35 = ~65.7%). Go release dashboard unchanged at unique-issue level. The only delta worth noting is **two minor patch releases from 2026-06-22 that the skill had not yet caught up on** — fixed in this cycle.

### Substantive findings

1. **Gin v1.13 milestone progress**: still **23/35 closed (~65.7%)**. Same as 2026-06-25 06:09 UTC.
2. **Go release dashboard**: unchanged at unique-issue level. Go 1.26.5 = 7 unique issues (header shows 8, double-counts #77800), Go 1.25.12 = 4 unique. Same set as 2026-06-25 06:09 UTC.
3. **Go stable releases**: unchanged — Go 1.26.4 / Go 1.25.11. No patch shipped.
4. **Go 1.27 RC1**: unchanged. `go1.27rc1` (tagged 2026-06-18T17:05:58Z) still latest. No RC2 yet.
5. **Validator floor-piercing risk** (from 2026-06-23 cycle's PR #4707): still active. `validator v10.30.3` still pins `x/crypto v0.52.0` + `x/sys v0.45.0`. No validator v10.30.4 yet.
6. **Zero new CVEs** in the past 6 hours affecting the Gin dependency tree (searched NVD for golang-keyword CVEs 2026-06-23 → 2026-06-25 — only Caddy 2.11.3 and Gogs 0.14.3 unrelated CVEs found).

### Refreshed dependency versions (minor bumps from 2026-06-22)

While checking for `proxy.golang.org/@latest` against the dependency table in this cycle, found two package versions that had been superseded but not yet propagated through the skill:

| Package | Old | New | Released | Severity |
|---|---|---|---|---|
| `github.com/redis/go-redis/v9` | v9.20.1 | **v9.21.0** | 2026-06-22 | Patch — drop-in, no API breakage. No CVE content in the release notes. |
| `gorm.io/gorm` | v1.31.1 | **v1.31.2** | 2026-06-22 | Patch — drop-in, no API breakage. No CVE content in the release notes. |

Refreshed in:
- `database.md` — "Updated from Research" section: go-redis line and GORM line updated; added a "Refreshed 2026-06-25" subsection.
- `versions.md` — dependency table row for go-redis/v9, the "Verified Versions" bullet list, and the cycle summary for 2026-06-22.

### Gin GitHub activity in the past 6 hours (since 2026-06-25 06:09 UTC)

Searched `https://api.github.com/repos/gin-gonic/gin/issues?state=all&since=2026-06-25T06:09:00Z`:
- **Zero closed** PRs/issues
- **Zero merged** PRs
- **One** open PR touched: #4689 (binding tryToSetValue refactor, cosmetic, updated 2026-06-25 06:42 UTC — **NOTE**: this update happened AFTER the prior cycle's 06:12 UTC commit so the prior cycle did not see it; the prior cycle's listing of #4689 as "updated 2026-06-25 06:42:31" came from the GitHub API snapshot at commit time. Author's own PR body describes the change as "completely negligible".)
- All other open PRs (#4660 data race, #4701 AbortedByHandler, #4705/#4693 AsciiJSON non-BMP, #4687 SkipMethodNotAllowed, #4712 BREAKING render refactor, #4714 ctx.Status no-body, etc.) unchanged from prior cycle.

### Gin master commits in past 6 hours

`https://api.github.com/repos/gin-gonic/gin/commits?per_page=20&since=2026-06-25T06:09:00Z` → **0 results**.

### No-change confirmations

- **Gin v1.12.0** (released 2026-02-28) still current stable. No v1.13 release tagged.
- **go-playground/validator/v10** — still at v10.30.3 (released 2026-05-29, no v10.30.4 yet). Floor-piercing risk unchanged.
- **golang.org/x/crypto** — still at v0.53.0 (released 2026-06-08, no newer release). v0.52.0 still pinned transitively by validator.
- **golang.org/x/sys** — still at v0.46.0 (released 2026-05-27, no newer release). v0.45.0 still pinned transitively by validator.
- **golang.org/x/text** — still at v0.38.0 / v0.37.0 (no newer release).
- **jackc/pgx/v5** — still at v5.10.0 (released 2026-06-03).
- **quic-go v0.60.0** — still current (released 2026-06-04).
- **golang-jwt/jwt/v5** — still at v5.3.1 (released 2026-01-28).
- **pressly/goose/v3** — still at v3.27.1 (released 2026-04-24).
- **ariga/atlas** — still at v1.2.0.
- **gin-contrib/sse** — still at v1.1.1 (released 2026-03-08).

### Action for agents

1. **`go get -u github.com/redis/go-redis/v9@v9.21.0 gorm.io/gorm@v1.31.2`** to pull the two minor refreshes into your Gin service go.mod. Both are drop-in, no migration needed.
2. **`golang.org/x/crypto v0.53.0` / `golang.org/x/sys v0.46.0` go.mod pin** still required for Gin v1.13 adopters (from PR #4707 finding in 2026-06-23 12:13 UTC cycle).
3. **Audit goroutine usage** in any Gin service for the data race pattern from PR #4660 (from 2026-06-25 06:09 UTC cycle).
4. **Go 1.26.5 / 1.25.12 still imminent** but not yet shipped. Re-verify `https://dev.golang.org/release` before next deploy.
5. **Go 1.27 RC1 still latest** — no RC2 yet. RC1 → RC2 cadence typically ~3 weeks, so RC2 expected ~2026-07-09 if the pattern holds. Watch `https://groups.google.com/g/golang-announce`.

### Sources for this update

- https://proxy.golang.org/github.com/redis/go-redis/v9/@latest (verified 2026-06-25 12:07 UTC — v9.21.0, released 2026-06-22)
- https://proxy.golang.org/gorm.io/gorm/@latest (verified 2026-06-25 12:07 UTC — v1.31.2, released 2026-06-22)
- https://proxy.golang.org/golang.org/x/crypto/@latest (verified 2026-06-25 12:07 UTC — v0.53.0 unchanged)
- https://proxy.golang.org/golang.org/x/sys/@latest (verified 2026-06-25 12:07 UTC — v0.46.0 unchanged)
- https://proxy.golang.org/github.com/go-playground/validator/v10/@v/list (verified 2026-06-25 12:07 UTC — latest still v10.30.3)
- https://go.dev/dl/?mode=json (re-verified 2026-06-25 12:07 UTC — `go1.27rc1` present, Go 1.26.4 current stable, Go 1.25.11 previous stable)
- https://raw.githubusercontent.com/golang/go/release-branch.go1.27/VERSION (re-verified 2026-06-25 12:07 UTC — `go1.27rc1`, time `2026-06-18T17:05:58Z`, unchanged)
- https://github.com/gin-gonic/gin/milestone/28 (v1.13 — re-verified 2026-06-25 12:07 UTC: **23/35 closed, ~65.7%**; same as 2026-06-25 06:09 UTC)
- https://api.github.com/repos/gin-gonic/gin/commits?per_page=20&since=2026-06-25T06:09:00Z (zero new master source commits in past 6 hours)
- https://api.github.com/repos/gin-gonic/gin/issues?state=all&since=2026-06-25T06:09:00Z (one open PR touched: #4689 at 06:42:31 UTC, no closures, no merges)
- https://api.github.com/repos/golang/go/milestones/438 (Go 1.25.12: open=4 closed=4, unchanged)
- https://api.github.com/repos/golang/go/milestones/439 (Go 1.26.5: open=7 closed=8, unchanged at unique-issue level)
- https://services.nvd.nist.gov/rest/json/cves/2.0?keywordSearch=golang&pubStartDate=2026-06-23T00:00:00.000 (no Gin/Go stdlib CVEs in past 3 days)


## Auto-update 2026-06-25 18:09 UTC (Cycle)

Six-hour cron cycle. Genuinely quiet — only ONE meaningful delta across all tracked dashboards, plus two already-noted items continue. No Gin master commits, no Gin merges, no new CVEs, no dependency releases.

### Deltas from prior cycle (2026-06-25 12:07 UTC)

1. **Go 1.26.5 release dashboard CL count: 7 → 8** (verified via `https://dev.golang.org/release` HTML scrape at 18:09 UTC, count-summary reads `<li><a href="#Go1.26.5">8 Go1.26.5</a></li>`). **New CL**: **`#80131` `cmd/link: peCreateExportFile generates invalid .def file when output name has trailing dot (c-shared)`** — Windows PE linker fix. When cross-compiling a Gin service from Linux/macOS to a Windows DLL (`go build -buildmode=c-shared -o mylib.dll.` with trailing-dot output name), the resulting `.def` file was malformed, breaking the `__declspec(dllexport)` table on the consumer side. Impact: **only matters for Gin services that ship Windows DLLs via `-buildmode=c-shared`**, which is unusual but not unheard of for Go-backed Windows plugins, COM out-of-proc servers, or Excel/Outlook add-ins. Standalone `.exe` and `.dll` (non-c-shared) builds are not affected. Fix will ship with Go 1.26.5 when the patch drops. The Go 1.26.5 unique-issue set is now: {`#77800`, `#79027`, `#79876`, `#79879`, `#79893`, `#80099`, `#80131`, (plus subref `79893`)}.
2. **Go 1.27 release-freeze day count remains 36** — we are still on June 25, 2026 UTC (prior cycle transitioned 35→36 at the Jun 25 boundary). Count advances to **37 at 2026-06-26 00:00 UTC**. RC1 (tagged 2026-06-18) is now 7 days old; RC2 cadence prediction unchanged at ~2026-07-09.
3. **Go 1.26.5 and Go 1.25.12 still NOT released** — both still pending. Patch releases have not been tagged on `https://go.dev/dl/?mode=json` since the prior 6-hour cycle.

### No-change confirmations

- **Gin v1.12.0** (released 2026-02-28) still current stable. No v1.13 release tagged.
- **Gin v1.13 milestone #28** still **23/35 closed (~65.7%)**, 12 open. **Due 2026-06-30** (5 days from this cycle). PRs open: #4674 url.PathUnescape, #4662 SSE helpers, #4569 H2CConfig, #4599 net/netip, #4543 whole-request binding, #4506 ResponseWriter.Unwrap, #4499 trailing slash, #4498 form nil-vs-empty, #4483 multi-write warning, #4712 BREAKING render/<format> opt-in, #4714 ctx.Status no-body flush, #4696 rune-boundary safety.
- **Go 1.27 RC1** still `go1.27rc1` (time `2026-06-18T17:05:58Z`), no RC2 tagged yet.
- **Go 1.26.4 / Go 1.25.11** still current/previous stable. No new patch release.
- **Go 1.25.12 pending CL count** still **4 CLs** (unchanged). Issue refs: {`#79026`, `#79875`, `#79878`, `#80098`}.
- **Gin master branch commits in past 6 hours**: **0** (verified `https://api.github.com/repos/gin-gonic/gin/commits?per_page=20&since=2026-06-25T12:09:00Z`).
- **Gin GitHub activity in past 6 hours**: 1 open PR touched (#4689 binding tryToSetValue refactor, cosmetic, no merge). Zero closures, zero merges.
- **Dependencies unchanged**: validator v10.30.3, quic-go v0.60.0, go-redis v9.21.0, gorm v1.31.2, golang-jwt v5.3.1, x/crypto v0.53.0, x/sys v0.46.0, x/net v0.56.0, goose v3.27.1, atlas v1.2.0, gin-contrib/cors v1.7.7, gin-contrib/zap v1.1.7, jackc/pgx/v5 v5.10.0, gin-contrib/sse v1.1.1.
- **Validator floor-piercing risk** (from 2026-06-23 PR #4707 finding): still active — `validator v10.30.3` transitively pins `x/crypto v0.52.0` and `x/sys v0.45.0`. No validator v10.30.4 yet.
- **No new CVEs** in past 3 days affecting the Gin dependency tree (`https://services.nvd.nist.gov/rest/json/cves/2.0?keywordSearch=golang&pubStartDate=2026-06-22T00:00:00.000` returned zero results).

### Action for agents

1. **`go get -u github.com/redis/go-redis/v9@v9.21.0 gorm.io/gorm@v1.31.2`** to pull the two minor refreshes into your Gin service go.mod. Both are drop-in, no migration needed (still recommended from prior cycle).
2. **`golang.org/x/crypto v0.53.0` / `golang.org/x/sys v0.46.0` go.mod pin** still required for Gin v1.13 adopters (from 2026-06-23 12:13 UTC PR #4707 finding).
3. **Gin v1.13 ships in 5 days** (due 2026-06-30). When it ships, audit for: (a) `c.ClientIP()` type change to `netip.Addr` (PR #4599), (b) trailing-slash behavior change (PR #4499), (c) `c.MsgPack`/`c.YAML`/`c.TOML`/`c.ProtoBuf`/`c.BSON` removal if PR #4712 lands.
4. **Go 1.26.5 imminent** — when it ships, the **`#80131` Windows PE linker fix** will unblock cross-compiled `-buildmode=c-shared` Windows DLL builds with trailing-dot output names. Most Gin services won't care; if you ship Windows plugins, the next security-patch Go upgrade will be load-bearing for you.
5. **Go 1.27 RC1 still latest** — no RC2 yet. Re-verify `https://go.dev/dl/?mode=json` before next deploy; RC2 expected ~2026-07-09 if cadence holds.

### Sources for this update

- https://dev.golang.org/release (verified 2026-06-25 18:09 UTC — 1.26.5: **8** CLs (was 7 prior cycle), 1.25.12: 4 CLs unchanged; counts read from page header `<li><a href="#Go1.26.5">8 Go1.26.5</a></li>` and `<li><a href="#Go1.25.12">4 Go1.25.12</a></li>`; new 1.26.5 CL `#80131 cmd/link: peCreateExportFile generates invalid .def file when output name has trailing dot (c-shared)` verified present in the 1.26.5 section)
- https://go.dev/dl/?mode=json (re-verified 2026-06-25 18:09 UTC — `go1.27rc1` present, Go 1.26.4 current stable, Go 1.25.11 previous stable, no patch shipped)
- https://raw.githubusercontent.com/golang/go/release-branch.go1.27/VERSION (re-verified 2026-06-25 18:09 UTC — `go1.27rc1`, time `2026-06-18T17:05:58Z`, unchanged)
- https://github.com/gin-gonic/gin/milestone/28 (v1.13 — re-verified 2026-06-25 18:09 UTC: **23/35 closed, ~65.7%**, due 2026-06-30; same as 2026-06-25 12:07 UTC)
- https://api.github.com/repos/gin-gonic/gin/commits?per_page=20&since=2026-06-25T12:09:00Z (zero new master source commits in past 6 hours)
- https://api.github.com/repos/gin-gonic/gin/issues?state=all&since=2026-06-25T12:09:00Z (one open PR touched: #4689, no closures, no merges)
- https://api.github.com/repos/golang/go/milestones/439 (Go 1.26.5: open=7 closed=8, unchanged at unique-issue level)
- https://api.github.com/repos/golang/go/milestones/438 (Go 1.25.12: open=4 closed=4, unchanged)
- https://services.nvd.nist.gov/rest/json/cves/2.0?keywordSearch=golang&pubStartDate=2026-06-22T00:00:00.000 (no Gin/Go stdlib CVEs in past 3 days)
- https://proxy.golang.org/github.com/go-playground/validator/v10/@latest (verified 2026-06-25 18:09 UTC — v10.30.3 unchanged)
- https://proxy.golang.org/golang.org/x/crypto/@latest (verified 2026-06-25 18:09 UTC — v0.53.0 unchanged)
- https://proxy.golang.org/golang.org/x/sys/@latest (verified 2026-06-25 18:09 UTC — v0.46.0 unchanged)
## Auto-update 2026-06-26 00:08 UTC (Cycle)

Six-hour cron cycle. ONE meaningful delta: **new CVE-2026-46602 / GO-2026-5062 published TODAY** in `golang.org/x/image/tiff` — skill's `golang.org/x/image` floor (v0.41.0) is **insufficient**, must be bumped to **v0.43.0+**. Otherwise quiet.

### Deltas from prior cycle (2026-06-25 18:09 UTC)

1. **NEW CVE-2026-46602 / GO-2026-5062** — `golang.org/x/image/tiff` unbounded tile-size memory allocation. Published 2026-06-25 (added to pkg.go.dev/vuln DB 2026-06-25 19:47 UTC). **CWE-789** (memory allocation with excessive size value). Fix: `golang.org/x/image` v0.43.0 (already published 2026-06-15 — fix predated CVE publication by 10 days, as is now common with Go's holding period). Adversary crafts a TIFF with an excessive `TileWidth` × `TileHeight` in the header; decoder allocates up to 4 GiB+ per tile. Same family as the older CVE-2026-33809 / CVE-2022-41727.
   - **Skill impact**: `golang.org/x/image v0.41.0+` floor (set 2026-05-29 for CVE-2026-42500 BMP panic) is **insufficient**. **New floor: `golang.org/x/image v0.43.0+`.**
   - **Action**: `go get golang.org/x/image@v0.43.0 && go mod tidy` on all Gin services that touch image uploads.
   - **Documented in**: `security.md` (new "CVE-2026-46602 / GO-2026-5062 — golang.org/x/image/tiff unbounded tile memory" section), `file-uploads.md` (Image Decode Safety section now covers both CVEs; Common Mistakes item #13 updated).
   - **Gin exposure**: TIFF-handling endpoints (medical imaging, scanner/PDF ingestion, scientific data, screenshot uploads), anything that calls `tiff.Decode` / `image.Decode` after `image.RegisterFormat("tiff", ...)`, OCR pipelines.
   - **Defense-in-depth added** to `file-uploads.md`: cap decoded pixel dimensions via `image.DecodeConfig` (reject if `Width × Height > 50M`).
2. **Go 1.27 release-freeze day count: 36 → 37** — UTC date rolled over to 2026-06-26 at 00:00 UTC. RC1 (tagged 2026-06-18) is now 8 days old; RC2 cadence prediction unchanged at ~2026-07-09.
3. **Go 1.26.5 and Go 1.25.12 still NOT released** — both still pending. Patch releases have not been tagged on `https://go.dev/dl/?mode=json` since the prior 6-hour cycle.

### No-change confirmations

- **Gin v1.12.0** (released 2026-02-28) still current stable. No v1.13 release tagged.
- **Gin v1.13 milestone #28** still **23/35 closed (~65.7%)**, 12 open. **Due 2026-06-30** (4 days from this cycle). Same open-PR list as 2026-06-25 18:09 UTC.
- **Go 1.27 RC1** still `go1.27rc1` (time `2026-06-18T17:05:58Z`), no RC2 tagged yet.
- **Go 1.26.4 / Go 1.25.11** still current/previous stable. No new patch release.
- **Go 1.26.5 pending CL count** still **8 CLs** (unchanged from 18:09 UTC cycle). Issue refs: {`#77800`, `#79027`, `#79876`, `#79879`, `#79893`, `#80099`, `#80131`, (plus subref `79893`)}.
- **Go 1.25.12 pending CL count** still **4 CLs** (unchanged). Issue refs: {`#79026`, `#79875`, `#79878`, `#80098`}.
- **Gin master branch commits in past 6 hours**: **0** (verified `https://api.github.com/repos/gin-gonic/gin/commits?per_page=20&since=2026-06-25T18:09:00Z`).
- **Gin GitHub activity in past 6 hours**: 0 open PRs touched, 0 closures, 0 merges.
- **Other dependencies unchanged**: validator v10.30.3, quic-go v0.60.0, go-redis v9.21.0, gorm v1.31.2, golang-jwt v5.3.1, x/crypto v0.53.0, x/sys v0.46.0, x/net v0.56.0, goose v3.27.1, atlas v1.2.0, gin-contrib/cors v1.7.7, gin-contrib/zap v1.1.7, jackc/pgx/v5 v5.10.0, gin-contrib/sse v1.1.1.
- **Validator floor-piercing risk** (from 2026-06-23 PR #4707 finding): still active — `validator v10.30.3` transitively pins `x/crypto v0.52.0` and `x/sys v0.45.0`. No validator v10.30.4 yet.
- **New non-x/image CVEs in past 6 hours**: **none**. `https://pkg.go.dev/vuln/list` shows no fresh `GO-2026-506x` entries since 5062.
- **Oracle Linux ELSA-2026-22121** (released 2026-06-23, advisory indexed 2026-06-24/25) is a re-packaging of upstream Go 1.26.3 fixes (CVE-2026-33811, CVE-2026-33814, CVE-2026-39817, CVE-2026-39819, CVE-2026-39820, CVE-2026-39823, CVE-2026-39825, CVE-2026-39826, CVE-2026-39836, CVE-2026-42499, CVE-2026-42501) — all already documented in skill's `security.md`. No new CVEs surfaced from this advisory.

### Action for agents

1. **`go get golang.org/x/image@v0.43.0 && go mod tidy`** on all Gin services handling image uploads. New CVE-2026-46602 is unauthenticated remote DoS via crafted TIFF; v0.41.0 floor is insufficient.
2. **`golang.org/x/crypto v0.53.0` / `golang.org/x/sys v0.46.0` go.mod pin** still required for Gin v1.13 adopters (from 2026-06-23 12:13 UTC PR #4707 finding).
3. **`go get -u github.com/redis/go-redis/v9@v9.21.0 gorm.io/gorm@v1.31.2`** to pull the two minor refreshes (from 2026-06-22 12:04 UTC cycle). Drop-in, no migration needed.
4. **Gin v1.13 ships in 4 days** (due 2026-06-30). When it ships, audit for: (a) `c.ClientIP()` type change to `netip.Addr` (PR #4599), (b) trailing-slash behavior change (PR #4499), (c) `c.MsgPack`/`c.YAML`/`c.TOML`/`c.ProtoBuf`/`c.BSON` removal if PR #4712 lands.
5. **Go 1.26.5 imminent** — the **`#80131` Windows PE linker fix** (c-shared trailing-dot output name) lands in this release. Affects only Gin services shipping Windows DLLs via `-buildmode=c-shared`.
6. **Go 1.27 RC1 still latest** — no RC2 yet. Re-verify `https://go.dev/dl/?mode=json` before next deploy.

### Sources for this update

- https://pkg.go.dev/vuln/list (verified 2026-06-26 00:08 UTC — `GO-2026-5062`/`CVE-2026-46602` newly listed for `golang.org/x/image/tiff`, fix in v0.43.0)
- https://pkg.go.dev/golang.org/x/image/tiff (verified 2026-06-26 00:08 UTC — `v0.43.0` published `Jun 15, 2026`)
- https://dev.golang.org/release (verified 2026-06-26 00:08 UTC — 1.26.5: 8 CLs unchanged, 1.25.12: 4 CLs unchanged)
- https://go.dev/dl/?mode=json (re-verified 2026-06-26 00:08 UTC — `go1.27rc1` present, Go 1.26.4 current stable, Go 1.25.11 previous stable, no patch shipped)
- https://raw.githubusercontent.com/golang/go/release-branch.go1.27/VERSION (re-verified 2026-06-26 00:08 UTC — `go1.27rc1`, time `2026-06-18T17:05:58Z`, unchanged)
- https://github.com/gin-gonic/gin/milestone/28 (v1.13 — re-verified 2026-06-26 00:08 UTC: **23/35 closed, ~65.7%**, due 2026-06-30; same as 2026-06-25 18:09 UTC)
- https://api.github.com/repos/gin-gonic/gin/commits?per_page=20&since=2026-06-25T18:09:00Z (zero new master source commits in past 6 hours)
- https://api.github.com/repos/gin-gonic/gin/issues?state=all&since=2026-06-25T18:09:00Z (no activity in past 6 hours)
- https://api.github.com/repos/golang/go/milestones/439 (Go 1.26.5: open=7 closed=8, unchanged)
- https://api.github.com/repos/golang/go/milestones/438 (Go 1.25.12: open=4 closed=4, unchanged)
- https://services.nvd.nist.gov/rest/json/cves/2.0?keywordSearch=golang&pubStartDate=2026-06-25T00:00:00.000 (only CVE-2026-46602 surfaced; CVE-2026-39822 still embargoed)
- https://proxy.golang.org/golang.org/x/image/@latest (verified 2026-06-26 00:08 UTC — v0.43.0 current; v0.41.0 confirmed insufficient for CVE-2026-46602)
- https://proxy.golang.org/github.com/go-playground/validator/v10/@latest (verified 2026-06-26 00:08 UTC — v10.30.3 unchanged; floor-piercing risk still active)
- https://proxy.golang.org/golang.org/x/crypto/@latest (verified 2026-06-26 00:08 UTC — v0.53.0 unchanged)
- https://proxy.golang.org/golang.org/x/sys/@latest (verified 2026-06-26 00:08 UTC — v0.46.0 unchanged)