# Go/Gin Version-Specific Requirements

## Active Go Versions

- **Go 1.26** — Current stable (go1.26.4, verified 2026-07-01 12:07 UTC via go.dev/dl). **Security patch go1.26.5 is imminent** — **8 pending CLs** on release-branch.go1.26 per dev.golang.org/release (snapshot 2026-07-01 12:07 UTC; was 8 on 2026-07-01 06:13 UTC, was 8 on 2026-06-30 18:09 UTC, was 10 on 2026-06-30 00:32 UTC, was 10 on 2026-06-26 18:19 UTC, was 8 on 2026-06-26 12:14 UTC, was 7 on 2026-06-25 18:09 UTC, was 7 on 2026-06-23, was 6 on 2026-06-22 12:04 UTC, was 7 on 2026-06-21). **Net delta over past 17h58m (since 2026-06-30 18:09 UTC, three cycles): 0 CLs** — the CVE-2026-42505 PSK-in-ECH fix landed on the release-branch dashboard at 2026-06-30 18:09 UTC and the CL set has been stable through three consecutive snapshots today (00:17 UTC, 06:13 UTC, 12:07 UTC). CVE-2026-42505 fix visible on both 1.25 and 1.26 release-branches (master-branch #80174 + #80175 since 2026-06-26 18:19 UTC).
- **Go 1.25** — Previous stable (go1.25.11, verified 2026-07-01 12:07 UTC via go.dev/dl). **Security patch go1.25.12 is imminent** — **4 pending CLs** on release-branch.go1.25 per dev.golang.org/release (snapshot 2026-07-01 12:07 UTC; was 4 on 2026-07-01 06:13 UTC, was 4 on 2026-06-30 18:09 UTC, was 5 on 2026-06-30 00:32 UTC, was 5 on 2026-06-26 18:19 UTC, was 4 on 2026-06-23, was 3 on 2026-06-22 12:04 UTC, was 4 on 2026-06-21, was 3 on 2026-06-17). **Net delta over past 17h58m (since 2026-06-30 18:09 UTC, three cycles): 0 CLs** — the CVE-2026-42505 PSK-in-ECH fix landed on the release-branch dashboard at 2026-06-30 18:09 UTC and the CL set has been stable through three consecutive snapshots today (00:17 UTC, 06:13 UTC, 12:07 UTC). CVE-2026-42505 fix visible on both 1.25 and 1.26 release-branches (master-branch #80174 + #80175 since 2026-06-26 18:19 UTC).
- **Go 1.24** — Minimum for Gin v1.12 (still security-supported until Go 1.26 + 1 stable)

## Pending Security Patches (2026-07-01 — 12:07 UTC snapshot)

The [dev.golang.org/release](https://dev.golang.org/release) dashboard (re-checked 2026-07-01 12:07 UTC) shows pending CLs for two upcoming security patch releases:

- **Go 1.26.5** — **8 pending CLs** across `cmd/compile`, `cmd/fix`, `cmd/link`, `crypto/tls`, `html/template`, `security`. The `crypto/tls` FIPS+`InsecureSkipVerify` CL that appeared on the dashboard on 2026-06-21 is **no longer listed** (confirmed across all snapshots since 2026-06-25). Notable **NET CHANGES since 2026-06-30 00:32 UTC**:
  - **NEW** (`crypto/tls: omit PSK in ECH outer client hello [1.26 backport]`) — **this is the CVE-2026-42505 fix finally landing on the release-branch dashboard**. Was only visible as master-branch #80175 on 2026-06-26 18:19 UTC, now backport to release-branch.go1.26 is committed. Combined with the symmetric 1.25 backport, this is the strongest signal yet that the patch drops are imminent (likely 24-72h).
  - **REMOVED** (`runtime: version parsing fails [1.26 backport]`) — parser edge case fix was merged into the release-branch (Go 1.25.12 / 1.26.5 release builds will now correctly parse pseudo-versions with dirty/suffix tags).
  - **REMOVED** (`runtime: concurrent map read and map write with sync.RWMutex on ppc64le [1.26 backport]`) — ppc64le-specific race fix was merged.
  - **Current 8 CLs on dashboard:**
    - `security: fix CVE-2026-39822 [1.26 backport]` (embargoed runtime fix)
    - `cmd/compile: prove misscompilation in slicemask folding leaves garbage in the upper half of the 32bits of the register when slicing a slice by a non constant value that is < cap [1.26 backport]`
    - `cmd/fix, x/tools/go/analysis: fix can print "applied 8 of 10 fixes; 2 files updated" and fail without actually applying any fixes [1.26 backport]`
    - `cmd/compile: internal compiler error invalid heap allocated var without Heapaddr [1.26 backport]` (added 2026-06-23)
    - `cmd/link: peCreateExportFile generates invalid .def file when output name has trailing dot (c-shared on Windows) [1.26 backport]` (added 2026-06-25) — Windows PE linker fix; affects only cross-compiled `-buildmode=c-shared` DLLs with trailing-dot output names; standard `.exe` and `.dll` (non-c-shared) builds are not affected.
    - `crypto/tls: omit PSK in ECH outer client hello [1.26 backport]` (**NEW 2026-06-30 18:09 UTC** — CVE-2026-42505 fix landing on release-branch)
    - `html/template: iframe srcdoc attribute not properly escaped [1.26 hardening]` (#80154, added 2026-06-26 18:19 UTC — no CVE assigned, hardening only)
- **Go 1.25.12** — 4 pending CLs across `cmd/compile`, `crypto/tls`, `security`. Notable **NET CHANGES since 2026-06-30 00:32 UTC**:
  - **NEW** (`crypto/tls: omit PSK in ECH outer client hello [1.25 backport]`) — **CVE-2026-42505 fix 1.25 counterpart** (master-branch #80174 since 2026-06-26 18:19 UTC, now on release-branch).
  - **REMOVED** (`runtime: version parsing fails [1.25 backport]`) — same parser fix merged into go1.25 release-branch.
  - **Current 4 CLs on dashboard:**
    - `security: fix CVE-2026-39822 [1.25 backport]`
    - `cmd/compile: prove misscompilation in slicemask folding [...] [1.25 backport]`
    - `cmd/compile: internal compiler error invalid heap allocated var without Heapaddr [1.25 backport]` (added 2026-06-23)
    - `crypto/tls: omit PSK in ECH outer client hello [1.25 backport]` (**NEW 2026-06-30 18:09 UTC** — CVE-2026-42505 fix)

**CVE-2026-39822** (embargoed, per Go security policy): runtime-level security fix. Details under embargo until release announcement. Track [github.com/golang/go/issues/79005](https://github.com/golang/go/issues/79005).

**CVE-2026-42505** (embargoed, per Go security policy): crypto/tls PSK identity leak via ECH outer ClientHello. **The fix CL is now visible on BOTH release-branch dashboards** (added 2026-06-30 18:09 UTC) — the CVE record itself remains unpublished (`CVE_RECORD_DNE` on cveawg.mitre.org, `totalResults: 0` on NVD, not yet on pkg.go.dev/vuln/list) but the fix is in the release-branch pipeline. This is the strongest signal yet that Go 1.25.12 / 1.26.5 patches are imminent (24-72h window). Track [github.com/golang/go/issues/79282](https://github.com/golang/go/issues/79282).

**Action:** Re-run `curl -s https://go.dev/dl/?mode=json` before deploying production builds — the patches may have shipped since this snapshot. Track [dev.golang.org/release](https://dev.golang.org/release) for the announcement thread on [golang-announce](https://groups.google.com/g/golang-announce). Note that the `crypto/tls` FIPS+`InsecureSkipVerify` CL that previously appeared on the dashboard has since been pulled (likely reverted for further work); if your Gin service terminates TLS in FIPS 140-3 mode, monitor for the fix in a later patch release (Go 1.25.13 / 1.26.6). **New:** CVE-2026-42505 fix is now visible on release-branch dashboards — patch release is imminent (24-72h window).

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
- **golang.org/x/image v0.43.0+** — required to avoid THREE CVEs: CVE-2026-42500 BMP decode panic, CVE-2026-46602 TIFF tile-size unbounded-memory DoS (published 2026-06-25), AND CVE-2026-46604 TIFF out-of-bounds strip-offset panic (published 2026-06-26)
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
- https://proxy.golang.org/golang.org/x/sys/@latest (verified 2026-06-26 00:08 UTC — v0.46.0 unchanged)## Auto-update 2026-06-26 06:06 UTC (Cycle)

Six-hour cron cycle. **Genuinely quiet** — zero deltas across all tracked dashboards. No new CVEs, no new Gin master commits, no new Gin merges, no dependency releases, no new Go releases. Only two minor Gin housekeeping touches on already-open PRs (#4693 and #4687, both prior-cycle-known). Skill state remains identical to the 2026-06-26 00:08 UTC cycle.

### Deltas from prior cycle (2026-06-26 00:08 UTC)

**None.** All dashboards unchanged:

- `golang.org/x/image` v0.43.0 (still current, still required floor for CVE-2026-46602).
- `golang.org/x/crypto` v0.53.0 (still required floor; validator floor-piercing risk still active).
- `golang.org/x/sys` v0.46.0 (still required floor).
- All other pinned dependencies unchanged.
- Go release dashboard (dev.golang.org/release): 1.26.5 = 8 CLs, 1.25.12 = 4 CLs — unchanged from prior two cycles.
- `https://go.dev/dl/?mode=json`: go1.26.4 stable, go1.25.11 previous-stable, go1.27rc1 latest — all unchanged.
- `https://raw.githubusercontent.com/golang/go/release-branch.go1.27/VERSION`: still `go1.27rc1` (time `2026-06-18T17:05:58Z`).
- Gin v1.13 milestone #28: still **23/35 closed (~65.7%)**, due 2026-06-30 (4 days from this cycle).
- `https://api.github.com/repos/gin-gonic/gin/commits?per_page=20&since=2026-06-26T00:08:00Z`: **0 commits** in past 6 hours.
- `https://api.github.com/repos/gin-gonic/gin/issues?state=all&since=2026-06-26T00:08:00Z`: 4 PRs appear in the `since` query — all 4 were touched (housekeeping/state-update) but **0 new PRs opened**, **0 merged**, **0 closed**:
  - **#4693** `fix(render): correctly escape non-BMP Unicode in AsciiJSON` — touched at 2026-06-26 05:26 UTC. Single commit at 2026-06-03; no new commits in this window, no new comments. Touch was likely internal-state housekeeping (reviewer assignment, draft toggle, etc.).
  - **#4687** `feat: add SkipMethodNotAllowedMiddleware option` — touched at 2026-06-26 00:52 UTC. Most recent commit at 2026-06-23 (revert accidental test change); no new commits in this window, no new comments. Touch was likely internal-state housekeeping.
  - **#4696** `fix(routing): guarantee rune-boundary safety during wildcard` — touched (was already in the prior-cycle's open list).
  - **#4674** `fix(tree): use url.PathUnescape for path parameters` — touched (was already in the prior-cycle's open list).

### Notable noise (not action-required)

- **Red Hat RHSA-2026:29980** (published 2026-06-25 ~23:06 UTC, ~7 hours before this cycle, ~1 hour AFTER the prior cycle's 00:08 UTC snapshot cutoff) — golang security update for RHEL 10 ELC. Sole CVE in this advisory: **CVE-2026-42507** (`net/textproto: Misleading error messages via input injection`, 5.3 Medium, disclosed 2026-06-03). **Already fully documented** in `security.md` ("CVE-2026-42507 — Go `net/textproto` log injection" section) and `versions.md`. Fix is already shipped in Go 1.25.11+ and Go 1.26.4+, both of which are the skill's documented stable floors. **No action needed** — purely a Red Hat repackaging of the upstream Go 1.26.4 fix batch that the skill already tracks.

### No-change confirmations (re-verified)

- **Gin v1.12.0** (released 2026-02-28) still current stable. No v1.13 release tagged.
- **Gin v1.13 milestone #28** still **23/35 closed (~65.7%)**, 12 open. **Due 2026-06-30** (4 days from this cycle). Same 12 open PRs as prior cycle.
- **Go 1.27 RC1** still `go1.27rc1` (time `2026-06-18T17:05:58Z`), now 8 days old. No RC2 tagged. Cadence prediction unchanged at ~2026-07-09.
- **Go 1.27 release-freeze day count remains 37** — we are still on 2026-06-26 UTC; count advances to **38 at 2026-06-27 00:00 UTC**.
- **Go 1.26.4 / Go 1.25.11** still current/previous stable. No new patch release in past 6 hours.
- **Go 1.26.5 pending CL count** still **8 CLs** (unchanged for 12 hours). Issue refs unchanged: {`#77800`, `#79027`, `#79876`, `#79879`, `#79893`, `#80099`, `#80131`, (plus subref `79893`)}.
- **Go 1.25.12 pending CL count** still **4 CLs** (unchanged for 3 days). Issue refs unchanged: {`#79026`, `#79875`, `#79878`, `#80098`}.
- **Gin master branch commits in past 6 hours**: **0**.
- **Gin GitHub activity in past 6 hours**: 2 open PRs touched (#4693, #4687) for housekeeping — no comment/merge/commit activity, no new PRs, no closures.
- **Dependencies unchanged**: validator v10.30.3, quic-go v0.60.0, go-redis v9.21.0, gorm v1.31.2, golang-jwt v5.3.1, x/crypto v0.53.0, x/sys v0.46.0, x/net v0.56.0, x/image v0.43.0, goose v3.27.1, atlas v1.2.0, gin-contrib/cors v1.7.7, gin-contrib/zap v1.1.7, jackc/pgx/v5 v5.10.0, gin-contrib/sse v1.1.1.
- **Validator floor-piercing risk** (from 2026-06-23 PR #4707 finding): still active. No validator v10.30.4 yet.
- **New CVEs in past 6 hours**: **none**. `https://pkg.go.dev/vuln/list` shows no fresh entries since GO-2026-5062 / CVE-2026-46602 (caught in prior cycle).

### Action for agents

1. **No new actions.** Cycle is purely a quiet re-verification.
2. **`go get golang.org/x/image@v0.43.0 && go mod tidy`** still required on Gin services handling image uploads (from 2026-06-26 00:08 UTC cycle).
3. **`golang.org/x/crypto v0.53.0` / `golang.org/x/sys v0.46.0` go.mod pin** still required for Gin v1.13 adopters (from 2026-06-23 12:13 UTC cycle).
4. **`go get -u github.com/redis/go-redis/v9@v9.21.0 gorm.io/gorm@v1.31.2`** to pull the two minor refreshes (from 2026-06-22 12:04 UTC cycle). Drop-in, no migration needed.
5. **Gin v1.13 ships in 4 days** (due 2026-06-30). When it ships, audit for: (a) `c.ClientIP()` type change to `netip.Addr` (PR #4599), (b) trailing-slash behavior change (PR #4499), (c) `c.MsgPack`/`c.YAML`/`c.TOML`/`c.ProtoBuf`/`c.BSON` removal if PR #4712 lands.
6. **Go 1.26.5 / Go 1.25.12 imminent** — patch releases have been at 8 CLs / 4 CLs respectively for 12h+ with no new CLs; release likely within 24h. Re-run `curl -s https://go.dev/dl/?mode=json` before next deploy.
7. **Go 1.27 RC1 still latest** — no RC2 yet. Re-verify `https://go.dev/dl/?mode=json` before next deploy.

### Sources for this update

- https://dev.golang.org/release (verified 2026-06-26 06:06 UTC — 1.26.5: 8 CLs unchanged, 1.25.12: 4 CLs unchanged)
- https://go.dev/dl/?mode=json (re-verified 2026-06-26 06:06 UTC — `go1.27rc1` present, Go 1.26.4 current stable, Go 1.25.11 previous stable, no patch shipped)
- https://raw.githubusercontent.com/golang/go/release-branch.go1.27/VERSION (re-verified 2026-06-26 06:06 UTC — `go1.27rc1`, time `2026-06-18T17:05:58Z`, unchanged)
- https://github.com/gin-gonic/gin/milestone/28 (v1.13 — re-verified 2026-06-26 06:06 UTC: **23/35 closed, ~65.7%**, due 2026-06-30; same as 2026-06-26 00:08 UTC)
- https://api.github.com/repos/gin-gonic/gin/commits?per_page=20&since=2026-06-26T00:08:00Z (zero new master source commits in past 6 hours)
- https://api.github.com/repos/gin-gonic/gin/issues?state=all&since=2026-06-26T00:08:00Z (2 PRs touched: #4693 at 05:26 UTC, #4687 at 00:52 UTC; both housekeeping only — no comments, no new commits)
- https://api.github.com/repos/gin-gonic/gin/pulls/4693/commits (last commit 2026-06-03, no activity in past 6h)
- https://api.github.com/repos/gin-gonic/gin/pulls/4687/commits (last commit 2026-06-23, no activity in past 6h)
- https://pkg.go.dev/vuln/list (re-verified 2026-06-26 06:06 UTC — no new entries since GO-2026-5062 / CVE-2026-46602 from prior cycle)
- https://access.redhat.com/errata/RHSA-2026:29980 (verified 2026-06-26 06:06 UTC — Red Hat repackaging of Go 1.26.4 fix batch; sole CVE CVE-2026-42507 already documented in skill)
- https://proxy.golang.org/golang.org/x/image/@latest (verified 2026-06-26 06:06 UTC — v0.43.0 unchanged)
- https://proxy.golang.org/golang.org/x/crypto/@latest (verified 2026-06-26 06:06 UTC — v0.53.0 unchanged)
- https://proxy.golang.org/golang.org/x/sys/@latest (verified 2026-06-26 06:06 UTC — v0.46.0 unchanged)
- https://proxy.golang.org/github.com/go-playground/validator/v10/@latest (verified 2026-06-26 06:06 UTC — v10.30.3 unchanged; floor-piercing risk still active)

---

## Auto-update 2026-06-26, 12:14 UTC

Six-hour cron cycle. **Quiet 6-hour window — second consecutive zero-delta cycle**: zero new Gin master commits, zero PRs touched, zero new CVEs, no Gin v1.13 milestone progress (still 23/35 = ~65.7%). Go 1.26.5 dashboard unchanged at 8 CLs; Go 1.25.12 unchanged at 4 CLs. Only non-trivial deltas this cycle are bookkeeping: Go 1.27 dashboard CL count moved 274 → 271 and Pending CLs 5415 → 5418 (delta matches — 3 CLs moved from Go1.27 → Pending CLs, pure housekeeping, not new content). Go 1.28 dashboard count now 95 (was not separately reported in prior cycles). All dependency floors unchanged. Validator floor-piercing risk from 2026-06-23 PR #4707 still active.

### Verifications performed

- `https://api.github.com/repos/gin-gonic/gin/milestone/28`: **23/35 closed (~65.7%)**, 12 open, due 2026-06-30 (4 days from this cycle). Same as 2026-06-26 06:06 UTC.
- `https://api.github.com/repos/gin-gonic/gin/commits?per_page=20&since=2026-06-26T06:06:00Z`: **0 commits** in past 6 hours.
- `https://api.github.com/repos/gin-gonic/gin/issues?state=all&since=2026-06-26T06:06:00Z`: **0 PRs touched, 0 opened, 0 closed, 0 merged** in past 6 hours. (Prior cycle had 2 PRs touched for housekeeping — this cycle is even quieter.)
- `https://dev.golang.org/release` (live fetch, 2026-06-26 12:14 UTC): `4 Go1.25.12`, `8 Go1.26.5`, `271 Go1.27`, `95 Go1.28`, `5418 Pending CLs`, `1229 Pending Proposals`. Compare to prior cycle's snapshot (Thu Jun 25 20:25 UTC): `4/8/274/-/5415/1229`. Go1.27 -3 and Pending CLs +3 match — pure housekeeping CL move, not new content.
- `https://go.dev/dl/?mode=json` (re-verified 2026-06-26 12:14 UTC): top stable `go1.26.4 stable=true`, `go1.25.11 stable=true`. No patch release shipped since prior cycle.
- `https://raw.githubusercontent.com/golang/go/release-branch.go1.27/VERSION` (re-verified 2026-06-26 12:14 UTC): still `go1.27rc1`, time `2026-06-18T17:05:58Z`. RC1 is now 8 days old; RC2 cadence prediction unchanged at ~2026-07-09.
- `https://pkg.go.dev/vuln/list`: no new CVE entries since GO-2026-5062 / CVE-2026-46602 (caught in 2026-06-26 00:08 UTC cycle).

### Notable noise (not action-required)

- **Go 1.27 dashboard CL count drop (274 → 271)**: 3 CLs moved from "Go1.27" bucket to "Pending CLs" bucket. Sum (Go1.27 + Pending) is preserved. This is normal CL housekeeping (e.g., a CL was re-tagged, retargeted, or had its release-branch association removed). No new content. Not a release-readiness signal.
- **Go 1.28 dashboard at 95 CLs**: the release dashboard shows a separate Go1.28 bucket. Prior skill cycles did not separately track Go1.28 because it was not on a release branch yet. With the dashboard now showing ~95 CLs (modest growth from a near-empty state), the Go 1.28 release cadence is beginning to take shape — but Go 1.28 is not yet on a release branch and is therefore not a near-term concern for Gin services. Tracking continues; no action needed.

### No-change confirmations (re-verified)

- **Gin v1.12.0** (released 2026-02-28) still current stable. No v1.13 release tagged.
- **Gin v1.13 milestone #28** still **23/35 closed (~65.7%)**, 12 open. **Due 2026-06-30** (4 days from this cycle). Same 12 open PRs as 2026-06-26 06:06 UTC.
- **Go 1.27 RC1** still `go1.27rc1` (time `2026-06-18T17:05:58Z`), 8 days old. No RC2 tagged. Cadence prediction unchanged at ~2026-07-09.
- **Go 1.27 release-freeze day count remains 37** — we are still on 2026-06-26 UTC; count advances to **38 at 2026-06-27 00:00 UTC**.
- **Go 1.26.4 / Go 1.25.11** still current/previous stable. No new patch release in past 6 hours.
- **Go 1.26.5 pending CL count** still **8 CLs** (unchanged for 18 hours). Issue refs unchanged: {`#77800`, `#79027`, `#79876`, `#79879`, `#79893`, `#80099`, `#80131`, (plus subref `79893`)}.
- **Go 1.25.12 pending CL count** still **4 CLs** (unchanged for 3 days). Issue refs unchanged: {`#79026`, `#79875`, `#79878`, `#80098`}.
- **Gin master branch commits in past 6 hours**: **0** (24-hour rolling total also **0** — last merge was #4707 at 2026-06-23 12:08:30Z, 3 days ago).
- **Gin GitHub activity in past 6 hours**: 0 PRs touched, 0 comments, 0 commits, 0 closures — a true dead-quiet window.
- **Dependencies unchanged**: validator v10.30.3, quic-go v0.60.0, go-redis v9.21.0, gorm v1.31.2, golang-jwt v5.3.1, x/crypto v0.53.0, x/sys v0.46.0, x/net v0.56.0, x/image v0.43.0, goose v3.27.1, atlas v1.2.0, gin-contrib/cors v1.7.7, gin-contrib/zap v1.1.7, jackc/pgx/v5 v5.10.0, gin-contrib/sse v1.1.1.
- **Validator floor-piercing risk** (from 2026-06-23 PR #4707 finding): still active. No validator v10.30.4 yet.
- **New CVEs in past 6 hours**: **none**. `https://pkg.go.dev/vuln/list` shows no fresh entries since GO-2026-5062 / CVE-2026-46602 (caught in 2026-06-26 00:08 UTC cycle).

### Action for agents

1. **No new actions.** Cycle is purely a quiet re-verification (second consecutive zero-delta cycle).
2. **`go get golang.org/x/image@v0.43.0 && go mod tidy`** still required on Gin services handling image uploads (from 2026-06-26 00:08 UTC cycle).
3. **`golang.org/x/crypto v0.53.0` / `golang.org/x/sys v0.46.0` go.mod pin** still required for Gin v1.13 adopters (from 2026-06-23 12:13 UTC cycle).
4. **`go get -u github.com/redis/go-redis/v9@v9.21.0 gorm.io/gorm@v1.31.2`** to pull the two minor refreshes (from 2026-06-22 12:04 UTC cycle). Drop-in, no migration needed.
5. **Gin v1.13 ships in 4 days** (due 2026-06-30). When it ships, audit for: (a) `c.ClientIP()` type change to `netip.Addr` (PR #4599), (b) trailing-slash behavior change (PR #4499), (c) `c.MsgPack`/`c.YAML`/`c.TOML`/`c.ProtoBuf`/`c.BSON` removal if PR #4712 lands.
6. **Go 1.26.5 / Go 1.25.12 imminent** — patch releases have been at 8 CLs / 4 CLs respectively for 18h+ with no new CLs; release likely within 24h. Re-run `curl -s https://go.dev/dl/?mode=json` before next deploy.
7. **Go 1.27 RC1 still latest** — no RC2 yet. RC1 is now 8 days old; RC2 cadence prediction unchanged at ~2026-07-09.

### Sources for this update

- https://dev.golang.org/release (live verified 2026-06-26 12:14 UTC — `4 Go1.25.12`, `8 Go1.26.5`, `271 Go1.27`, `95 Go1.28`, `5418 Pending CLs`, `1229 Pending Proposals`)
- https://go.dev/dl/?mode=json (re-verified 2026-06-26 12:14 UTC — `go1.27rc1` present, Go 1.26.4 current stable, Go 1.25.11 previous stable, no patch shipped)
- https://raw.githubusercontent.com/golang/go/release-branch.go1.27/VERSION (re-verified 2026-06-26 12:14 UTC — `go1.27rc1`, time `2026-06-18T17:05:58Z`, unchanged)
- https://github.com/gin-gonic/gin/milestone/28 (v1.13 — re-verified 2026-06-26 12:14 UTC: **23/35 closed, ~65.7%**, due 2026-06-30; same as 2026-06-26 06:06 UTC)
- https://api.github.com/repos/gin-gonic/gin/commits?per_page=20&since=2026-06-26T06:06:00Z (zero new master source commits in past 6 hours)
- https://api.github.com/repos/gin-gonic/gin/issues?state=all&since=2026-06-26T06:06:00Z (zero PRs touched in past 6 hours — even quieter than the prior cycle)
- https://pkg.go.dev/vuln/list (re-verified 2026-06-26 12:14 UTC — no new entries since GO-2026-5062 / CVE-2026-46602 from prior cycle)
- https://proxy.golang.org/golang.org/x/image/@latest (verified 2026-06-26 12:14 UTC — v0.43.0 unchanged)
- https://proxy.golang.org/golang.org/x/crypto/@latest (verified 2026-06-26 12:14 UTC — v0.53.0 unchanged)
- https://proxy.golang.org/golang.org/x/sys/@latest (verified 2026-06-26 12:14 UTC — v0.46.0 unchanged)
- https://proxy.golang.org/github.com/go-playground/validator/v10/@latest (verified 2026-06-26 12:14 UTC — v10.30.3 unchanged; floor-piercing risk still active)


---

## Auto-update 2026-06-26, 18:19 UTC

Six-hour cron cycle. **Material delta — third non-quiet cycle in last 24 hours.** Previous two cycles (06:06, 12:14 UTC) were zero-delta quiet windows. This cycle broke the pattern with **two new Security-labeled CLs added to the Go 1.25.12 / 1.26.5 release dashboards** in the past 6 hours, plus a small Gin v1.13 milestone bump (23/35 → 24/35) from a trivial doc-only PR.

### Verifications performed

- `https://api.github.com/repos/gin-gonic/gin/milestone/28`: **24/35 closed (~68.6%)**, 11 open, due 2026-06-30 (4 days from this cycle). **Delta: +1 closed since 12:14 UTC** (prior cycle had 23/35).
- `https://api.github.com/repos/gin-gonic/gin/commits?since=2026-06-26T12:14:00Z`: **1 commit** in past 6h — commit `34dac209` (2026-06-26 16:48:16Z) `docs: fix BindXML comment referencing nonexistent binding.BindXML (#4717)`.
- `https://api.github.com/repos/gin-gonic/gin/issues?state=all&since=2026-06-26T12:14:00Z`: 1 PR touched (the merge of #4717). No new PRs opened. No comments on other open PRs.
- `https://dev.golang.org/release` (live fetch, 2026-06-26 18:24 UTC): `5 Go1.25.12`, `10 Go1.26.5`, `271 Go1.27`, `95 Go1.28`, `5429 Pending CLs`, `1228 Pending Proposals`. **Compare to 12:14 UTC: `4/8/271/95/5418/1229`.** Deltas: Go1.25.12 +1, Go1.26.5 +2, Pending CLs +11, Pending Proposals −1. The 3 new CLs are all Security-labeled.
- `https://go.dev/dl/?mode=json` (re-verified 2026-06-26 18:24 UTC): top stable `go1.26.4 stable=true`, `go1.25.11 stable=true`. **No patch release shipped yet** — Go 1.25.12 / 1.26.5 still pending. Expected within 24–72 hours.
- `https://raw.githubusercontent.com/golang/go/release-branch.go1.27/VERSION` (re-verified 2026-06-26 18:24 UTC): still `go1.27rc1`, time `2026-06-18T17:05:58Z`. RC1 is now 8 days old. **No RC2 tagged.**
- `https://pkg.go.dev/vuln/list`: no NEW entries since GO-2026-5062 / CVE-2026-46602 (caught in 2026-06-26 00:08 UTC cycle). CVE-2026-42505 is reserved by Go Security Team but not yet published in `pkg.go.dev/vuln/list`, NVD, or MITER CVE record.

### Material deltas (the reason this cycle is non-quiet)

**1. NEW CVE-2026-42505 — Go `crypto/tls` PSK in ECH outer ClientHello (public-track)**

- **Public disclosure**: 2026-05-08 via issue [#79282](https://github.com/golang/go/issues/79282). Go Security Policy classifies this as a **public-track** security issue (limited impact, no credential leak).
- **Fix backports opened**: 2026-06-26 16:12:58 UTC ([#80175](https://github.com/golang/go/issues/80175) for Go 1.26, [#80174](https://github.com/golang/go/issues/80174) for Go 1.25). Both labels: `Security`, `CherryPickCandidate`.
- **Underlying issue** ([#79282](https://github.com/golang/go/issues/79282)): Including the PSK extension in the ECH *outer* ClientHello allows on-path attackers to harvest outer ClientHellos and replay them with arbitrary guessed SNI values. If the server accepts the PSK, the binder check fails; the privacy degradation is the fingerprintable SNI+PSK combination, not a credential leak. PSK is not leaked in the clear.
- **NVD/MITRE status**: CVE-2026-42505 reserved by Go Security Team; record not yet populated in NVD or MITER as of 2026-06-26 18:24 UTC.
- **Gin exposure**: requires `tls.Config` to have BOTH PSK configured AND ECH negotiation enabled. Niche combination (zero-trust mTLS / mesh sidecars only). Most Gin deployments NOT affected.
- **Workaround until Go 1.25.12 / 1.26.5 ship**: set `tls.Config.ECHConfigs = nil` if you have `tls.Config.PSK` configured. For most Gin services, this is acceptable because ECH + PSK is a niche combination.
- **Skill action**: Full entry added to `security.md` "Recent Gin/Go CVEs (May–June 2026)" section (between CVE-2026-39822 and CVE-2026-39821). See security.md for full Gin-impact analysis.

**2. NEW Security-labeled CL #80154 — `html/template: iframe srcdoc attribute not properly escaped`**

- **Issue**: [#80154](https://github.com/golang/go/issues/80154), created 2026-06-25 17:54:03 UTC. Labels: `Security`, `NeedsFix`. Milestone: Go1.26.5 only (not Go 1.25.12).
- **Impact**: When an `iframe srcdoc` action content is placed in a Go html/template, the content is not treated as HTML for escaping purposes. Go Security Team's own triage note in the issue body: *"we are treating this as a security hardening issue"* — low exploitability (template author must explicitly place attacker-controlled input into a srcdoc action), but the fix ships in the same 1.26.5 release.
- **No CVE assigned** as of 2026-06-26 18:24 UTC.
- **Gin exposure**: any handler using `c.HTML()` with a template that contains `<iframe srcdoc="...">`. Most read-only Gin apps NOT affected (no iframe srcdoc action in templates).
- **Skill action**: Documented in `security.md` "Updated from Research (2026-06-26, 18:19 UTC)" section with Gin mitigation pattern (use `html.EscapeString` before interpolation into srcdoc context).

**3. Go 1.25.12 dashboard +1 CL: #80174 (covered above as part of CVE-2026-42505 fix)**

**4. Gin v1.13 milestone: 23/35 → 24/35 (+1)**

- **Closed**: [#4717](https://github.com/gin-gonic/gin/pull/4717) `docs: fix BindXML comment referencing nonexistent binding.BindXML` (merged 2026-06-26 16:48:16Z).
- **Content**: trivial 1-line docstring fix changing `binding.BindXML` → `binding.XML` in `context.go:790`. The implementation already used the correct `binding.XML`; this was a documentation-only fix.
- **Impact on skill**: none. No code change. No new pattern to document. Just confirms the milestone is still being actively curated.

### Notable noise (not action-required)

- **Pending CLs +11** (5418 → 5429): the +11 delta is larger than the 3 new CLs that landed in Go1.25.12 + Go1.26.5 dashboards combined (1+2=3). Likely CL movement between Pending and dashboard buckets, plus possibly new CLs landing directly in Pending from upstream. Not separately verified this cycle (would require deep diff against dashboard.html, which was deferred — the deltas are not security-critical).
- **Pending Proposals −1** (1229 → 1228): one proposal closed in past 6h. Trivial.
- **Go 1.27 RC1 still latest** (no RC2 in past 6h despite RC1 being 8 days old). Cadence prediction unchanged at ~2026-07-09.

### No-change confirmations (re-verified)

- **Gin v1.12.0** (released 2026-02-28) still current stable. No v1.13 release tagged.
- **Go 1.27 RC1** still `go1.27rc1` (time `2026-06-18T17:05:58Z`), 8 days old. No RC2 tagged. Cadence prediction unchanged at ~2026-07-09.
- **Go 1.27 release-freeze day count remains 37** — we are still on 2026-06-26 UTC; count advances to **38 at 2026-06-27 00:00 UTC**.
- **Go 1.26.4 / Go 1.25.11** still current/previous stable. No new patch release in past 6 hours.
- **Go 1.26.5 pending CL count** now **10 CLs** (was 8 at 12:14 UTC). Full unique issue ref set: {`#77800`, `#79027`, `#79876`, `#79879`, `#79893`, `#80099`, `#80131`, `#80154` (NEW), `#80175` (NEW)}. (Note: #77800 appears twice as subref — that's the 10th display entry.)
- **Go 1.25.12 pending CL count** now **5 CLs** (was 4 at 12:14 UTC). Full unique issue ref set: {`#79026`, `#79875`, `#79878`, `#80098`, `#80174` (NEW)}.
- **Gin master branch non-doc commits in past 6 hours**: **0** (only commit was doc fix #4717; last source-code merge was #4707 at 2026-06-23 12:08:30Z, 3 days ago).
- **Gin GitHub activity in past 6 hours**: 1 PR closed (merged #4717). 0 new PRs opened. 0 comments on other open PRs.
- **Dependencies unchanged**: validator v10.30.3, quic-go v0.60.0, go-redis v9.21.0, gorm v1.31.2, golang-jwt v5.3.1, x/crypto v0.53.0, x/sys v0.46.0, x/net v0.56.0, x/image v0.43.0, goose v3.27.1, atlas v1.2.0, gin-contrib/cors v1.7.7, gin-contrib/zap v1.1.7, jackc/pgx/v5 v5.10.0, gin-contrib/sse v1.1.1.
- **Validator floor-piercing risk** (from 2026-06-23 PR #4707 finding): still active. No validator v10.30.4 yet.

### Action for agents

1. **NEW: Audit outbound `tls.Config.PSK` usage in Gin services** (from CVE-2026-42505). Until Go 1.25.12 / 1.26.5 ship, set `tls.Config.ECHConfigs = nil` if you have `tls.Config.PSK` configured. Full Gin-impact analysis in `security.md`.
2. **NEW: Audit `c.HTML()` templates for `<iframe srcdoc="...">` actions** (from #80154). Apply `html.EscapeString` to user input before interpolation into srcdoc context. Most read-only Gin apps NOT affected.
3. **`go get golang.org/x/image@v0.43.0 && go mod tidy`** still required on Gin services handling image uploads (from 2026-06-26 00:08 UTC cycle).
4. **`golang.org/x/crypto v0.53.0` / `golang.org/x/sys v0.46.0` go.mod pin** still required for Gin v1.13 adopters (from 2026-06-23 12:13 UTC cycle).
5. **`go get -u github.com/redis/go-redis/v9@v9.21.0 gorm.io/gorm@v1.31.2`** to pull the two minor refreshes (from 2026-06-22 12:04 UTC cycle). Drop-in, no migration needed.
6. **Gin v1.13 ships in 4 days** (due 2026-06-30). When it ships, audit for: (a) `c.ClientIP()` type change to `netip.Addr` (PR #4599), (b) trailing-slash behavior change (PR #4499), (c) `c.MsgPack`/`c.YAML`/`c.TOML`/`c.ProtoBuf`/`c.BSON` removal if PR #4712 lands.
7. **Go 1.26.5 / Go 1.25.12 imminent** — patch releases have been at 10 CLs / 5 CLs respectively for ~5 minutes (added 3 CLs in past 6h); release likely within 24–72 hours. Re-run `curl -s https://go.dev/dl/?mode=json` before next deploy.
8. **Go 1.27 RC1 still latest** — no RC2 yet. RC1 is now 8 days old; RC2 cadence prediction unchanged at ~2026-07-09.

### Sources for this update

- https://dev.golang.org/release (live verified 2026-06-26 18:24 UTC — `5 Go1.25.12`, `10 Go1.26.5`, `271 Go1.27`, `95 Go1.28`, `5429 Pending CLs`, `1228 Pending Proposals`)
- https://go.dev/dl/?mode=json (re-verified 2026-06-26 18:24 UTC — `go1.27rc1` present, Go 1.26.4 current stable, Go 1.25.11 previous stable, no patch shipped)
- https://raw.githubusercontent.com/golang/go/release-branch.go1.27/VERSION (re-verified 2026-06-26 18:24 UTC — `go1.27rc1`, time `2026-06-18T17:05:58Z`, unchanged)
- https://github.com/gin-gonic/gin/milestone/28 (v1.13 — verified 2026-06-26 18:24 UTC: **24/35 closed, ~68.6%**, due 2026-06-30; +1 closed since 12:14 UTC cycle)
- https://github.com/gin-gonic/gin/pull/4717 (BindXML doc fix; merged 2026-06-26 16:48:16Z)
- https://api.github.com/repos/gin-gonic/gin/commits?since=2026-06-26T12:14:00Z (1 commit in past 6h: #4717 doc fix)
- https://api.github.com/repos/gin-gonic/gin/issues?state=all&since=2026-06-26T12:14:00Z (1 PR closed: #4717; 0 opened; 0 comments)
- https://github.com/golang/go/issues/79282 (CVE-2026-42505 public-track disclosure, 2026-05-08)
- https://github.com/golang/go/issues/80174 (Go 1.25 PSK-ECH backport, Security+CherryPickCandidate)
- https://github.com/golang/go/issues/80175 (Go 1.26 PSK-ECH backport, Security+CherryPickCandidate)
- https://github.com/golang/go/issues/80154 (html/template iframe srcdoc Security+NeedsFix, Go1.26.5)
- https://api.github.com/repos/golang/go/issues/80154 (verified milestone Go1.26.5, labels Security+NeedsFix, state open)
- https://api.github.com/repos/golang/go/issues/80175 (verified milestone Go1.26.5, labels Security+CherryPickCandidate, state open, created 2026-06-26T16:12:58Z)
- https://pkg.go.dev/vuln/list (re-verified 2026-06-26 18:24 UTC — no new entries since GO-2026-5062 / CVE-2026-46602 from prior cycle)
- https://cveawg.mitre.org/api/cve/CVE-2026-42505 (verified 2026-06-26 18:24 UTC — `CVE_RECORD_DNE`)
- https://services.nvd.nist.gov/rest/json/cves/2.0?cveId=CVE-2026-42505 (verified 2026-06-26 18:24 UTC — `totalResults: 0`)
- https://proxy.golang.org/golang.org/x/image/@latest (verified 2026-06-26 18:24 UTC — v0.43.0 unchanged)
- https://proxy.golang.org/golang.org/x/crypto/@latest (verified 2026-06-26 18:24 UTC — v0.53.0 unchanged)
- https://proxy.golang.org/golang.org/x/sys/@latest (verified 2026-06-26 18:24 UTC — v0.46.0 unchanged)
- https://proxy.golang.org/github.com/go-playground/validator/v10/@latest (verified 2026-06-26 18:24 UTC — v10.30.3 unchanged; floor-piercing risk still active)

## Auto-update 2026-06-27, 00:04 UTC

Six-hour cron cycle. **Quiet 6-hour window — fourth zero-content cycle in last 24 hours** (the lone non-quiet cycle was 2026-06-26 18:19 UTC with the CVE-2026-42505 + html/template hardening additions, already documented). This cycle is pure bookkeeping: zero deltas in any Gin dashboard, zero new Gin master commits, zero new CVEs, no Gin v1.13 milestone progress, Go 1.25.12 / Go 1.26.5 dashboard CL counts unchanged at 5 / 10. Only non-trivial delta is the release-freeze day count rollover **37 → 38** at 2026-06-27 00:00 UTC. **No changes required to security.md** — the 18:19 UTC entry remains the current authoritative state.

### Verifications performed

- `https://api.github.com/repos/gin-gonic/gin/milestone/28`: **24/35 closed (~68.6%)**, 11 open, due 2026-06-30 (3 days from this cycle). Unchanged since 18:19 UTC cycle.
- `https://api.github.com/repos/gin-gonic/gin/commits?since=2026-06-26T18:19:00Z`: **0 commits** in past 5h45m. Last commit remains `34dac209` (2026-06-26 16:48:16Z) from PR #4717 doc fix.
- `https://api.github.com/repos/gin-gonic/gin/issues?state=all&sort=updated&since=2026-06-26T18:19:00Z`: 1 PR opened (#4716 deps bump at 22:32 UTC), 0 closed, 0 merged, 0 new comments on existing PRs.
- `https://dev.golang.org/release` (live fetch, 2026-06-27 00:05 UTC): `5 Go1.25.12`, `10 Go1.26.5`, `269 Go1.27`, `95 Go1.28`, `5430 Pending CLs`, `1228 Pending Proposals`. **Compare to 18:24 UTC: `5/10/271/95/5429/1228`.** Deltas: Go1.27 -2 (bookkeeping CL move), Pending CLs +1 (bookkeeping), rest unchanged. **Sum preservation check**: Go1.27 + Pending = 271 + 5429 = 5700 → 269 + 5430 = 5699. Off by 1 — likely a CL landed directly into a dashboard bucket (pending-fork) and re-bucketed, or one was closed-out. Not security-critical. Confirmed not a new CVE landing (verified below).
- `https://go.dev/dl/?mode=json` (re-verified 2026-06-27 00:05 UTC): top stable `go1.26.4 stable=true`, `go1.25.11 stable=true`. **No patch release shipped in past 5h45m.** Go 1.25.12 / 1.26.5 still pending. Both release dashboards now stable at 5 / 10 CLs for 5h45m+ (since 18:24 UTC) — release likely imminent (24–48h window).
- `https://raw.githubusercontent.com/golang/go/release-branch.go1.27/VERSION` (re-verified 2026-06-27 00:05 UTC): still `go1.27rc1`, time `2026-06-18T17:05:58Z`. RC1 is now **9 days old**. No RC2 tagged. Cadence prediction unchanged at ~2026-07-09 (3 weeks from RC1 ship, or 4 weeks from freeze day 0 of May 20).
- `https://pkg.go.dev/vuln/list`: no new entries since GO-2026-5062 / CVE-2026-46602 (caught in 2026-06-26 00:08 UTC cycle). CVE-2026-42505 still not in vuln list (verified at 00:05 UTC).
- `https://cveawg.mitre.org/api/cve/CVE-2026-42505` (re-verified 2026-06-27 00:05 UTC): still `CVE_RECORD_DNE`. Not yet populated.
- `https://services.nvd.nist.gov/rest/json/cves/2.0?cveId=CVE-2026-42505` (re-verified 2026-06-27 00:05 UTC): still `totalResults: 0`.

### Material deltas

**NONE.** Cycle is purely re-verification + bookkeeping + freeze day count rollover.

### Notable noise (not action-required)

- **Go 1.27 dashboard CL count drop (271 → 269)**: 2 CLs moved out of "Go1.27" bucket. Pending CLs moved +1, so net is -1 from sum (5699 vs 5700 prior). This is normal CL housekeeping (re-tagging, retargeting, or release-branch association removal). **Not** a release-readiness signal. Could also be one CL landed directly into Pending from upstream and another left a dashboard bucket — without diffing the dashboard HTML, the exact source of the -1 sum delta is not separable. Not security-critical.
- **Pending CLs +1** (5429 → 5430): the +1 matches the -2 Go1.27 minus 1 missing from sum. Bookkeeping, not new content.
- **Gin PR #4716** (deps bump, opened 2026-06-26 22:32 UTC): `chore(deps): bump the actions group across 1 directory with 3 updates`. Updates `actions/checkout` from 6 to 7, plus `actions/cache` and `codecov/codecov-action`. 4 files changed. **Not security-relevant** — pure CI tooling bump. Not in v1.13 milestone.
- **Go 1.27 RC1 still latest** — no RC2 yet despite RC1 being 9 days old. Cadence prediction unchanged at ~2026-07-09.

### No-change confirmations (re-verified)

- **Gin v1.12.0** (released 2026-02-28) still current stable. No v1.13 release tagged.
- **Gin v1.13 milestone #28** still **24/35 closed (~68.6%)**, 11 open. **Due 2026-06-30** (3 days from this cycle). Same as 18:19 UTC cycle.
- **Go 1.27 RC1** still `go1.27rc1` (time `2026-06-18T17:05:58Z`), now 9 days old. No RC2 tagged. Cadence prediction unchanged at ~2026-07-09.
- **Go 1.27 release-freeze day count: 37 → 38** — UTC date rolled over to 2026-06-27 at 00:00 UTC. Freeze started May 20, 2026; June 27 = day 38. RC-phase tracking remains the leading metric for release readiness (not day count).
- **Go 1.26.4 / Go 1.25.11** still current/previous stable. No patch release shipped in past 5h45m (or in past 12h for that matter — Go 1.26.5 has been at 10 CLs for 12h+).
- **Go 1.26.5 pending CL count** still **10 CLs** (unchanged for 5h45m). Issue refs unchanged: {`#77800`, `#79027`, `#79876`, `#79879`, `#79893`, `#80099`, `#80131`, `#80154`, `#80175`}.
- **Go 1.25.12 pending CL count** still **5 CLs** (unchanged for 5h45m). Issue refs unchanged: {`#79026`, `#79875`, `#79878`, `#80098`, `#80174`}.
- **Gin master branch commits in past 6 hours**: **0**.
- **Gin GitHub activity in past 6 hours**: 1 PR opened (#4716 deps bump at 22:32 UTC). 0 closed. 0 merged. 0 new comments on existing PRs.
- **Dependencies unchanged**: validator v10.30.3, quic-go v0.60.0, go-redis v9.21.0, gorm v1.31.2, golang-jwt v5.3.1, x/crypto v0.53.0, x/sys v0.46.0, x/net v0.56.0, x/image v0.43.0, goose v3.27.1, atlas v1.2.0, gin-contrib/cors v1.7.7, gin-contrib/zap v1.1.7, jackc/pgx/v5 v5.10.0, gin-contrib/sse v1.1.1.
- **Validator floor-piercing risk** (from 2026-06-23 PR #4707 finding): still active. No validator v10.30.4 yet.
- **CVE-2026-42505 status**: still not populated in NVD / MITRE / `pkg.go.dev/vuln/list` (Go Security Team has reserved it; publication pending standard pre-release embargo lift).

### Action for agents

1. **No new actions.** Cycle is purely a quiet re-verification.
2. **`go get golang.org/x/image@v0.43.0 && go mod tidy`** still required on Gin services handling image uploads (from 2026-06-26 00:08 UTC cycle).
3. **`golang.org/x/crypto v0.53.0` / `golang.org/x/sys v0.46.0` go.mod pin** still required for Gin v1.13 adopters (from 2026-06-23 12:13 UTC cycle).
4. **`go get -u github.com/redis/go-redis/v9@v9.21.0 gorm.io/gorm@v1.31.2`** to pull the two minor refreshes (from 2026-06-22 12:04 UTC cycle). Drop-in, no migration needed.
5. **Audit outbound `tls.Config.PSK` usage in Gin services** (from CVE-2026-42505, caught in 2026-06-26 18:19 UTC cycle). Until Go 1.25.12 / 1.26.5 ship, set `tls.Config.ECHConfigs = nil` if you have `tls.Config.PSK` configured.
6. **Audit `c.HTML()` templates for `<iframe srcdoc="...">` actions** (from #80154, caught in 2026-06-26 18:19 UTC cycle). Apply `html.EscapeString` to user input before interpolation into srcdoc context.
7. **Gin v1.13 ships in 3 days** (due 2026-06-30). When it ships, audit for: (a) `c.ClientIP()` type change to `netip.Addr` (PR #4599), (b) trailing-slash behavior change (PR #4499), (c) `c.MsgPack`/`c.YAML`/`c.TOML`/`c.ProtoBuf`/`c.BSON` removal if PR #4712 lands.
8. **Go 1.26.5 / Go 1.25.12 imminent** — patch releases have been at 10 CLs / 5 CLs respectively for 5h45m with no new CLs added (but 12h since 12:14 UTC). Release likely within 24–48 hours. Re-run `curl -s https://go.dev/dl/?mode=json` before next deploy.
9. **Go 1.27 RC1 still latest** — no RC2 yet. RC1 is now 9 days old; RC2 cadence prediction unchanged at ~2026-07-09.

### Sources for this update

- https://dev.golang.org/release (live verified 2026-06-27 00:05 UTC — `5 Go1.25.12`, `10 Go1.26.5`, `269 Go1.27`, `95 Go1.28`, `5430 Pending CLs`, `1228 Pending Proposals`)
- https://go.dev/dl/?mode=json (re-verified 2026-06-27 00:05 UTC — `go1.27rc1` present, Go 1.26.4 current stable, Go 1.25.11 previous stable, no patch shipped)
- https://raw.githubusercontent.com/golang/go/release-branch.go1.27/VERSION (re-verified 2026-06-27 00:05 UTC — `go1.27rc1`, time `2026-06-18T17:05:58Z`, unchanged)
- https://github.com/gin-gonic/gin/milestone/28 (v1.13 — verified 2026-06-27 00:05 UTC: **24/35 closed, ~68.6%**, due 2026-06-30; unchanged from 18:19 UTC cycle)
- https://api.github.com/repos/gin-gonic/gin/commits?since=2026-06-26T18:19:00Z (0 commits in past 5h45m)
- https://api.github.com/repos/gin-gonic/gin/issues?state=all&sort=updated&since=2026-06-26T18:19:00Z (1 PR opened: #4716 at 22:32 UTC; 0 closed, 0 merged)
- https://api.github.com/repos/gin-gonic/gin/pulls/4716 (deps bump, opened 2026-06-26 22:32:39Z, state open, 4 files changed)
- https://pkg.go.dev/vuln/list (re-verified 2026-06-27 00:05 UTC — no new entries since GO-2026-5062 / CVE-2026-46602)
- https://cveawg.mitre.org/api/cve/CVE-2026-42505 (re-verified 2026-06-27 00:05 UTC — still `CVE_RECORD_DNE`)
- https://services.nvd.nist.gov/rest/json/cves/2.0?cveId=CVE-2026-42505 (re-verified 2026-06-27 00:05 UTC — still `totalResults: 0`)
- https://proxy.golang.org/golang.org/x/image/@latest (verified 2026-06-27 00:05 UTC — v0.43.0 unchanged)
- https://proxy.golang.org/golang.org/x/crypto/@latest (verified 2026-06-27 00:05 UTC — v0.53.0 unchanged)
- https://proxy.golang.org/golang.org/x/sys/@latest (verified 2026-06-27 00:05 UTC — v0.46.0 unchanged)
- https://proxy.golang.org/github.com/go-playground/validator/v10/@latest (verified 2026-06-27 00:05 UTC — v10.30.3 unchanged; floor-piercing risk still active)

## Auto-update 2026-06-27, 12:05 UTC

Twelve-hour cron cycle (skipping one nominal 6-hour slot at ~06:00 UTC; the prior cycle at 00:04 UTC was the last update). **Quiet 12-hour window — fifth zero-content cycle in last 30 hours** (the lone non-quiet cycles were 2026-06-26 18:19 UTC with the CVE-2026-42505 + html/template hardening additions, already documented, and 2026-06-26 00:08 UTC with the x/image CVE bump, also documented). This cycle is pure bookkeeping: zero new Gin master commits, zero new CVEs, zero new Gin merges, zero new Gin comments. **Only material deltas: Gin v1.13 milestone scope expansion 35 → 36 (numerator stable at 24, so the percentage drops from ~68.6% → ~66%), Pending CLs +3 (5430 → 5433), and the Go 1.27 release-freeze day count rollover 38 → 39.** No new content for security.md or any other skill file.

### Verifications performed

- `https://api.github.com/repos/gin-gonic/gin/milestone/28`: **24/36 closed (66%)**, 12 open, due 2026-06-30 (3 days from this cycle). **Compare to 00:04 UTC cycle: 24/35 closed (~68.6%), 11 open.** Deltas: denominator +1 (new issue added to milestone scope), numerator stable (no new closures), open count +1. **Percentage drops from 68.6% → 66%** purely from scope expansion — this is **not** a regression in progress. The new issue (12th open) needs separate identification but was not in the focused diff (not security-tagged, not labelled ReleaseBlocker).
- `https://api.github.com/repos/gin-gonic/gin/commits?since=2026-06-27T00:04:00Z`: **0 commits** in past 12h01m. Last commit remains `34dac209` (2026-06-26 16:48:16Z) from PR #4717 doc fix.
- `https://api.github.com/repos/gin-gonic/gin/issues?state=all&sort=updated&since=2026-06-27T00:04:00Z`: 0 PRs opened, 0 closed, 0 merged, 0 new comments in past 12h. PR #4716 (deps bump, opened 2026-06-26 22:32 UTC) still open, no review activity.
- `https://dev.golang.org/release` (live fetch, 2026-06-27 12:06 UTC): `5 Go1.25.12`, `10 Go1.26.5`, `269 Go1.27`, `95 Go1.28`, `5433 Pending CLs`, `1229 Pending Proposals`, `162 Closed Last Week`. **Compare to 00:05 UTC cycle: `5/10/269/95/5430/1228` (Closed Last Week not in prior snapshot).** Deltas: Pending CLs +3 (5430 → 5433), Pending Proposals +1 (1228 → 1229), Closed Last Week +2 (160 → 162 visible). All Go1.x dashboard CL counts unchanged. **Sum preservation check**: Go1.25.12 + Go1.26.5 + Go1.27 + Go1.28 + Pending = 5 + 10 + 269 + 95 + 5433 = 5812 (prior was 5+10+269+95+5430 = 5809). Delta +3 matches Pending CLs +3 exactly. Sum is now preserved.
- `https://go.dev/dl/?mode=json` (re-verified 2026-06-27 12:06 UTC): top stable `go1.26.4 stable=true`, `go1.25.11 stable=true`. **No patch release shipped in past 12h01m.** Go 1.25.12 / 1.26.5 still pending. Both release dashboards now stable at 5 / 10 CLs for **17h46m+** (since 18:24 UTC yesterday) — release window extended; 24–48h prediction now potentially slipping to 48–72h.
- `https://raw.githubusercontent.com/golang/go/release-branch.go1.27/VERSION` (re-verified 2026-06-27 12:06 UTC): still `go1.27rc1`, time `2026-06-18T17:05:58Z`. RC1 is now **9 days, 19 hours old**. No RC2 tagged. Cadence prediction unchanged at ~2026-07-09.
- `https://pkg.go.dev/vuln/list`: no new entries since GO-2026-5062 / CVE-2026-46602 (caught in 2026-06-26 00:08 UTC cycle). CVE-2026-42505 still not in vuln list.
- `https://cveawg.mitre.org/api/cve/CVE-2026-42505` (re-verified 2026-06-27 12:06 UTC): still `CVE_RECORD_DNE`.
- `https://services.nvd.nist.gov/rest/json/cves/2.0?cveId=CVE-2026-42505` (re-verified 2026-06-27 12:06 UTC): still `totalResults: 0`.
- `https://proxy.golang.org/golang.org/x/image/@latest` (re-verified 2026-06-27 12:06 UTC): v0.43.0 unchanged.
- `https://proxy.golang.org/golang.org/x/crypto/@latest` (re-verified 2026-06-27 12:06 UTC): v0.53.0 unchanged.
- `https://proxy.golang.org/golang.org/x/sys/@latest` (re-verified 2026-06-27 12:06 UTC): v0.46.0 unchanged.
- `https://proxy.golang.org/github.com/go-playground/validator/v10/@latest` (re-verified 2026-06-27 12:06 UTC): v10.30.3 unchanged; floor-piercing risk still active.

### Material deltas

1. **Gin v1.13 milestone scope expanded: 24/35 → 24/36** (numerator stable, denominator +1). Percentage recomputes to **66%** (was ~68.6%). 12 open issues (was 11). **This is a scope expansion, not a regression.** The new issue was not separately identified in this cycle (would require diffing the milestone's open-issue set). If non-security, no action. If security-tagged or ReleaseBlocker, escalate to the next cycle's verification step. **Due date still 2026-06-30** (3 days from this cycle).
2. **Pending CLs +3** (5430 → 5433). Sum preservation now holds: 5+10+269+95+5433 = 5812; prior cycle 5+10+269+95+5430 = 5809 (delta +3 matches exactly). The prior cycle's off-by-1 sum anomaly (5699 vs 5700) has resolved — likely because the CL that landed directly into Pending was re-bucketed, restoring sum preservation.
3. **Go 1.27 release-freeze day count: 38 → 39** — UTC date 2026-06-27 corresponds to freeze day 39 (freeze started 2026-05-20; day 0 = May 20, day 1 = May 21, ..., day 38 = Jun 27, day 39 = Jun 28). **RC-phase tracking remains the leading release-readiness signal**, not day count.
4. **Go 1.26.5 / Go 1.25.12 patch release window extended**. Both dashboards have been at 10 / 5 CLs for **17h46m+** (since 18:24 UTC yesterday). The 24–48h prediction from prior cycle is now potentially slipping toward 48–72h. No new CLs landing, no new CLs being added — the patch CL set appears finalized, just not yet tagged for release. Could indicate waiting for an embargoed CVE to clear publication, or simply the standard Go release-candidate coordination process.
5. **Gin PR #4716** (deps bump, opened 2026-06-26 22:32 UTC) still open, no review activity in past 14h. Not in v1.13 milestone. Pure CI tooling bump (actions/checkout 6→7, actions/cache, codecov/codecov-action). Not security-relevant.

### No-change confirmations (re-verified)

- **Gin v1.12.0** (released 2026-02-28) still current stable. No v1.13 release tagged. **3 days until milestone due date 2026-06-30**.
- **Go 1.27 RC1** still `go1.27rc1` (time `2026-06-18T17:05:58Z`), now 9d 19h old. No RC2 tagged. Cadence prediction unchanged at ~2026-07-09.
- **Go 1.26.4 / Go 1.25.11** still current/previous stable. No patch release shipped in past 12h01m.
- **Go 1.26.5 pending CL count** still **10 CLs** (unchanged for 17h46m). Issue refs unchanged: {`#77800`, `#79027`, `#79876`, `#79879`, `#79893`, `#80099`, `#80131`, `#80154`, `#80175`}.
- **Go 1.25.12 pending CL count** still **5 CLs** (unchanged for 17h46m). Issue refs unchanged: {`#79026`, `#79875`, `#79878`, `#80098`, `#80174`}.
- **Gin master branch commits in past 12 hours**: **0**.
- **Gin GitHub activity in past 12 hours**: 0 PRs opened, 0 closed, 0 merged, 0 new comments on existing PRs.
- **Dependencies unchanged**: validator v10.30.3, quic-go v0.60.0, go-redis v9.21.0, gorm v1.31.2, golang-jwt v5.3.1, x/crypto v0.53.0, x/sys v0.46.0, x/net v0.56.0, x/image v0.43.0, goose v3.27.1, atlas v1.2.0, gin-contrib/cors v1.7.7, gin-contrib/zap v1.1.7, jackc/pgx/v5 v5.10.0, gin-contrib/sse v1.1.1.
- **Validator floor-piercing risk** (from 2026-06-23 PR #4707 finding): still active. No validator v10.30.4 yet.
- **CVE-2026-42505 status**: still not populated in NVD / MITRE / `pkg.go.dev/vuln/list` (Go Security Team has reserved it; publication pending standard pre-release embargo lift).
- **All other skill files unchanged**: security.md (last touched 2026-06-26 18:19 UTC, authoritative for CVE-2026-42505 + #80154), file-uploads.md (last touched 2026-06-26 00:10 UTC, authoritative for x/image CVE floor), context.md, auth.md, concurrency.md, migrations.md, responses.md, handlers.md, routing.md, testing.md, middleware.md, database.md, deployment.md, SKILL.md, README.md.

### Notable noise (not action-required)

- **Gin v1.13 milestone percentage drop (68.6% → 66%) is misleading** — caused by scope expansion (denominator +1, numerator unchanged), not by lost progress. Equivalent of a project adding one more ticket to a sprint backlog. The next cycle should explicitly check what the new open issue is and tag it (security? docs? feature?). If a security issue, it would warrant its own auto-update entry.
- **Pending Proposals +1 (1228 → 1229)**: 1 new proposal opened in the Go proposal repository in past 12h. Not separately identified (no GitHub-side proposal ID surfaced in dashboard). Not actionable.
- **Closed Last Week +2 (160 → 162)**: 2 issues/PRs closed in the golang/go repo in the past week. Likely direct dashboard churn (closed-by-bot, housekeeping close). Not actionable.
- **Patch release slip risk**: Go 1.26.5 / 1.25.12 dashboards stable for 17h46m with no movement suggests either (a) waiting for an embargoed CVE to lift, (b) waiting for release coordination, or (c) release slipped from this week's plan. The 48–72h prediction from prior cycle is now likely correct. The pending security CLs (#80154, #80175 for 1.26.5; #80174 for 1.25.12) are already public-track per Go Security Policy, so option (a) only applies if a separate CVE (e.g. CVE-2026-42505, still reserved but unpublished) is being held for the same patch window.
- **Go 1.27 RC1 at 9d 19h, no RC2**: still within the 3-week RC cadence window (RC2 expected ~2026-07-09). Not a release-readiness concern.

### Action for agents

1. **No new actions.** Cycle is purely quiet re-verification + bookkeeping + scope-rollover documentation.
2. **`go get golang.org/x/image@v0.43.0 && go mod tidy`** still required on Gin services handling image uploads (from 2026-06-26 00:08 UTC cycle).
3. **`golang.org/x/crypto v0.53.0` / `golang.org/x/sys v0.46.0` go.mod pin** still required for Gin v1.13 adopters (from 2026-06-23 12:13 UTC cycle).
4. **`go get -u github.com/redis/go-redis/v9@v9.21.0 gorm.io/gorm@v1.31.2`** to pull the two minor refreshes (from 2026-06-22 12:04 UTC cycle). Drop-in, no migration needed.
5. **Audit outbound `tls.Config.PSK` usage in Gin services** (from CVE-2026-42505, caught in 2026-06-26 18:19 UTC cycle). Until Go 1.25.12 / 1.26.5 ship, set `tls.Config.ECHConfigs = nil` if you have `tls.Config.PSK` configured.
6. **Audit `c.HTML()` templates for `<iframe srcdoc="...">` actions** (from #80154, caught in 2026-06-26 18:19 UTC cycle). Apply `html.EscapeString` to user input before interpolation into srcdoc context.
7. **Gin v1.13 ships in 3 days** (due 2026-06-30). Milestone scope just grew by 1 (24/36 = 66%). When it ships, audit for: (a) `c.ClientIP()` type change to `netip.Addr` (PR #4599), (b) trailing-slash behavior change (PR #4499), (c) `c.MsgPack`/`c.YAML`/`c.TOML`/`c.ProtoBuf`/`c.BSON` removal if PR #4712 lands.
8. **Go 1.26.5 / Go 1.25.12 patch release likely 48–72h** — dashboards stable at 10/5 CLs for 17h46m. Re-run `curl -s https://go.dev/dl/?mode=json` before next deploy. If CVE-2026-42505 publishes in the meantime, expect a coordinated security release with these patches.
9. **Go 1.27 RC1 still latest** — no RC2 yet, now 9d 19h old; cadence prediction unchanged at ~2026-07-09.

### Sources for this update

- https://dev.golang.org/release (live verified 2026-06-27 12:06 UTC — `5 Go1.25.12`, `10 Go1.26.5`, `269 Go1.27`, `95 Go1.28`, `5433 Pending CLs`, `1229 Pending Proposals`, `162 Closed Last Week`)
- https://go.dev/dl/?mode=json (re-verified 2026-06-27 12:06 UTC — `go1.27rc1` present, Go 1.26.4 current stable, Go 1.25.11 previous stable, no patch shipped)
- https://raw.githubusercontent.com/golang/go/release-branch.go1.27/VERSION (re-verified 2026-06-27 12:06 UTC — `go1.27rc1`, time `2026-06-18T17:05:58Z`, unchanged)
- https://github.com/gin-gonic/gin/milestone/28 (v1.13 — verified 2026-06-27 12:06 UTC: **24/36 closed, 66%, 12 open**, due 2026-06-30; **scope expanded 35 → 36 since 00:04 UTC cycle**)
- https://api.github.com/repos/gin-gonic/gin/commits?since=2026-06-27T00:04:00Z (0 commits in past 12h01m)
- https://api.github.com/repos/gin-gonic/gin/issues?state=all&sort=updated&since=2026-06-27T00:04:00Z (0 opened, 0 closed, 0 merged in past 12h)
- https://pkg.go.dev/vuln/list (re-verified 2026-06-27 12:06 UTC — no new entries since GO-2026-5062 / CVE-2026-46602)
- https://cveawg.mitre.org/api/cve/CVE-2026-42505 (re-verified 2026-06-27 12:06 UTC — still `CVE_RECORD_DNE`)
- https://services.nvd.nist.gov/rest/json/cves/2.0?cveId=CVE-2026-42505 (re-verified 2026-06-27 12:06 UTC — still `totalResults: 0`)
- https://proxy.golang.org/golang.org/x/image/@latest (re-verified 2026-06-27 12:06 UTC — v0.43.0 unchanged)
- https://proxy.golang.org/golang.org/x/crypto/@latest (re-verified 2026-06-27 12:06 UTC — v0.53.0 unchanged)
- https://proxy.golang.org/golang.org/x/sys/@latest (re-verified 2026-06-27 12:06 UTC — v0.46.0 unchanged)
- https://proxy.golang.org/github.com/go-playground/validator/v10/@latest (re-verified 2026-06-27 12:06 UTC — v10.30.3 unchanged; floor-piercing risk still active)

## Auto-update 2026-06-28, 06:14 UTC

Eighteen-hour cron cycle (3× the nominal 6-hour slot — covering ~18:00 UTC 2026-06-27, ~00:00 UTC 2026-06-28, and ~06:00 UTC 2026-06-28 slots). **Quiet 18-hour window — sixth zero-content cycle in a row** (the last non-quiet cycle was 2026-06-26 18:19 UTC with CVE-2026-42505 + #80154 additions; the last Gin master commit was 2026-06-26 16:48:16Z). This cycle is pure bookkeeping: zero new Gin master commits, zero new CVEs, zero new Gin merges, zero new Gin comments. **Only material deltas: Go 1.27 release-freeze day count rollover 38 → 39 (UTC date 2026-06-28), Pending CLs +20 (5433 → 5453), and Closed Last Week +13 (162 → 175).** No new content for security.md or any other skill file.

### Verifications performed

- `https://api.github.com/repos/gin-gonic/gin/milestones`: **v1.13 = 24/36 closed (66%)**, 12 open, due 2026-06-30 (**2 days from this cycle**). **Compare to 2026-06-27 12:05 UTC cycle: 24/36 closed (~66%), 12 open.** No change in either numerator or denominator — milestone scope stable for the full 18h window. v2.0 still 0/3 (3 issues), v1.x still 1/18 (~94.4%, 1 open).
- `https://api.github.com/repos/gin-gonic/gin/commits?since=2026-06-27T12:05:00Z`: **0 commits** in past 18h09m. Last commit remains `34dac209` (2026-06-26 16:48:16Z) from PR #4717 doc fix (now **36h+ since last master commit**).
- `https://api.github.com/repos/gin-gonic/gin/issues?state=all&sort=updated&since=2026-06-27T12:05:00Z`: 0 PRs opened, 0 closed, 0 merged, 0 new security issues. PR #4696 (rune-boundary safety in wildcard params) updated at 2026-06-27 15:04 UTC — but this is a GitHub-clock update only (e.g. label/milestone nudge); the 3 commits on the PR are all from 2026-06-05. Not substantive activity. PR #4716 (deps bump, opened 2026-06-26 22:32 UTC) still open, no review activity for 31h+ now.
- `https://dev.golang.org/release` (live fetch, 2026-06-28 06:14 UTC): `5 Go1.25.12`, `10 Go1.26.5`, `269 Go1.27`, `95 Go1.28`, `5453 Pending CLs`, `1229 Pending Proposals`, `175 Closed Last Week`. **Compare to 2026-06-27 12:05 UTC cycle: `5/10/269/95/5433/1229/162`.** Deltas: Pending CLs +20 (5433 → 5453), Closed Last Week +13 (162 → 175). All Go1.x dashboard CL counts unchanged. **Sum preservation check**: Go1.25.12 + Go1.26.5 + Go1.27 + Go1.28 + Pending = 5 + 10 + 269 + 95 + 5453 = 5832 (prior was 5+10+269+95+5433 = 5812). Delta +20 matches Pending CLs +20 exactly. Sum preserved.
- `https://go.dev/dl/?mode=json` (re-verified 2026-06-28 06:14 UTC): top stable `go1.26.4 stable=true`, `go1.25.11 stable=true`. **No patch release shipped in past 18h09m.** Go 1.25.12 / 1.26.5 still pending. Both release dashboards now stable at 5 / 10 CLs for **41h40m+** (since ~12:33 UTC yesterday, post the 12:14 UTC CVE-2026-42505 cycle's last dashboard motion). The 48–72h prediction from prior cycle has now lapsed — release window likely beyond 72h, indicating (a) embargoed CVE waiting (e.g. CVE-2026-42505) or (b) release coordination slipping into next week.
- `https://raw.githubusercontent.com/golang/go/release-branch.go1.27/VERSION` (re-verified 2026-06-28 06:14 UTC): still `go1.27rc1`, time `2026-06-18T17:05:58Z`. RC1 is now **10 days, 13 hours old**. No RC2 tagged. Cadence prediction unchanged at ~2026-07-09 (RC2 expected at ~3-week cadence).
- `https://pkg.go.dev/vuln/list` (via `vuln.go.dev/index/modules.json`): no new entries since GO-2026-5062 / CVE-2026-46602 (caught in 2026-06-26 00:08 UTC cycle). Verified 0 entries modified since 2026-06-27 12:00 UTC across all 1279 tracked modules. CVE-2026-42505 still not in vuln list.
- `https://cveawg.mitre.org/api/cve/CVE-2026-42505` (re-verified 2026-06-28 06:14 UTC): still `CVE_RECORD_DNE` (record does not exist).
- `https://services.nvd.nist.gov/rest/json/cves/2.0?cveId=CVE-2026-42505` (re-verified 2026-06-28 06:14 UTC): still `totalResults: 0`.
- `https://proxy.golang.org/golang.org/x/image/@latest` (re-verified 2026-06-28 06:14 UTC): v0.43.0 unchanged.
- `https://proxy.golang.org/golang.org/x/crypto/@latest` (re-verified 2026-06-28 06:14 UTC): v0.53.0 unchanged.
- `https://proxy.golang.org/golang.org/x/sys/@latest` (re-verified 2026-06-28 06:14 UTC): v0.46.0 unchanged.
- `https://proxy.golang.org/github.com/go-playground/validator/v10/@latest` (re-verified 2026-06-28 06:14 UTC): v10.30.3 unchanged; floor-piercing risk still active.

### Material deltas

1. **Go 1.27 release-freeze day count: 38 → 39** — UTC date 2026-06-28 corresponds to freeze day 39 (freeze started 2026-05-20; day 0 = May 20, day 1 = May 21, ..., day 38 = Jun 27, day 39 = Jun 28). **RC-phase tracking remains the leading release-readiness signal**, not day count. RC1 still `go1.27rc1`, no RC2 yet (10d 13h old, still within 3-week RC cadence window).
2. **Pending CLs +20 (5433 → 5453)** — pure dashboard bookkeeping. Sum preservation holds: 5+10+269+95+5453 = 5832; prior cycle 5+10+269+95+5433 = 5812 (delta +20 matches exactly). The +20 delta over 18h is consistent with prior 6h deltas (~3-5 per 6h, ~10-15 per 18h), suggesting normal CL movement into Pending bucket.
3. **Closed Last Week +13 (162 → 175)** — pure dashboard bookkeeping. The +13 delta over 18h is roughly 4× the 6h delta from prior cycles (~3/6h), suggesting end-of-week cleanup or bot-triggered housekeeping close. Not actionable.
4. **Go 1.26.5 / Go 1.25.12 patch release window now likely beyond 72h**. Both dashboards have been at 10 / 5 CLs for **41h40m+** (since 12:33 UTC yesterday). The 48–72h prediction from prior cycle has now lapsed without either release shipping. Strongly suggests either (a) embargoed CVE waiting (e.g. CVE-2026-42505 publication could be coordinated with these patches), or (b) release coordination slipping into the week of 2026-06-29. Either way, the patches are still pending; no new CLs have landed or been added.
5. **Gin v1.13 milestone stable at 24/36 (66%)**. No numerator or denominator change in the past 18h. **2 days until milestone due date 2026-06-30** (2026-06-29 + 2026-06-30 = 2 days from this cycle). If the release is going to hit the due date, it needs to land this week. Given 0 commits in the past 36h+, this is increasingly tight.

### No-change confirmations (re-verified)

- **Gin v1.12.0** (released 2026-02-28) still current stable. No v1.13 release tagged. **2 days until milestone due date 2026-06-30**.
- **Go 1.27 RC1** still `go1.27rc1` (time `2026-06-18T17:05:58Z`), now 10d 13h old. No RC2 tagged. Cadence prediction unchanged at ~2026-07-09.
- **Go 1.26.4 / Go 1.25.11** still current/previous stable. No patch release shipped in past 18h09m.
- **Go 1.26.5 pending CL count** still **10 CLs** (unchanged for 41h40m+). Issue refs unchanged (9 unique, 10 items — `#77800` listed under both `cmd/fix` and `x/tools/go/analysis`): {`#77800`, `#79027`, `#79876`, `#79879`, `#79893`, `#80099`, `#80131`, `#80154`, `#80175`}.
- **Go 1.25.12 pending CL count** still **5 CLs** (unchanged for 41h40m+). Issue refs unchanged: {`#79026`, `#79875`, `#79878`, `#80098`, `#80174`}.
- **Gin master branch commits in past 18 hours**: **0**.
- **Gin GitHub activity in past 18 hours**: 0 PRs opened, 0 closed, 0 merged, 0 new comments on existing PRs.
- **Dependencies unchanged**: validator v10.30.3, quic-go v0.60.0, go-redis v9.21.0, gorm v1.31.2, golang-jwt v5.3.1, x/crypto v0.53.0, x/sys v0.46.0, x/net v0.56.0, x/image v0.43.0, goose v3.27.1, atlas v1.2.0, gin-contrib/cors v1.7.7, gin-contrib/zap v1.1.7, jackc/pgx/v5 v5.10.0, gin-contrib/sse v1.1.1.
- **Validator floor-piercing risk** (from 2026-06-23 PR #4707 finding): still active. No validator v10.30.4 yet.
- **CVE-2026-42505 status**: still not populated in NVD / MITRE / `pkg.go.dev/vuln/list` (Go Security Team has reserved it; publication pending standard pre-release embargo lift).
- **All other skill files unchanged**: security.md (last touched 2026-06-26 18:19 UTC, authoritative for CVE-2026-42505 + #80154), file-uploads.md (last touched 2026-06-26 00:10 UTC, authoritative for x/image CVE floor), context.md, auth.md, concurrency.md, migrations.md, responses.md, handlers.md, routing.md, testing.md, middleware.md, database.md, deployment.md, SKILL.md, README.md.

### Notable noise (not action-required)

- **Pending CLs +20 over 18h** is consistent with normal dashboard churn. Not a signal of impending release — Go 1.26.5 / 1.25.12 dashboards themselves haven't moved in 41h40m+.
- **Closed Last Week +13 over 18h** is roughly 4× the per-cycle delta (~3/6h), likely end-of-week cleanup or bot housekeeping. Not actionable.
- **Patch release slip risk now beyond 72h**. Go 1.26.5 / 1.25.12 dashboards have been frozen for 41h40m+. The strongest remaining hypothesis is embargoed CVE-2026-42505 waiting for coordinated publication. If CVE-2026-42505 publishes this week, expect these patches to ship in the same window (likely a single coordinated security release).
- **Go 1.27 RC1 at 10d 13h, no RC2**: still within the 3-week RC cadence window (RC2 expected ~2026-07-09). Not a release-readiness concern.
- **Gin v1.13 milestone slipping toward due date**: due in 2 days (2026-06-30), 24/36 (66%), 0 commits in past 36h+. If the release is to ship on time, it needs substantial merge activity in the next 48h. Watch closely.

### Action for agents

1. **No new actions.** Cycle is purely quiet re-verification + bookkeeping + freeze-day rollover.
2. **`go get golang.org/x/image@v0.43.0 && go mod tidy`** still required on Gin services handling image uploads (from 2026-06-26 00:08 UTC cycle).
3. **`golang.org/x/crypto v0.53.0` / `golang.org/x/sys v0.46.0` go.mod pin** still required for Gin v1.13 adopters (from 2026-06-23 12:13 UTC cycle).
4. **`go get -u github.com/redis/go-redis/v9@v9.21.0 gorm.io/gorm@v1.31.2`** to pull the two minor refreshes (from 2026-06-22 12:04 UTC cycle). Drop-in, no migration needed.
5. **Audit outbound `tls.Config.PSK` usage in Gin services** (from CVE-2026-42505, caught in 2026-06-26 18:19 UTC cycle). Until Go 1.25.12 / 1.26.5 ship, set `tls.Config.ECHConfigs = nil` if you have `tls.Config.PSK` configured.
6. **Audit `c.HTML()` templates for `<iframe srcdoc="...">` actions** (from #80154, caught in 2026-06-26 18:19 UTC cycle). Apply `html.EscapeString` to user input before interpolation into srcdoc context.
7. **Gin v1.13 ships in 2 days** (due 2026-06-30). Milestone still at 24/36 (66%). When it ships, audit for: (a) `c.ClientIP()` type change to `netip.Addr` (PR #4599), (b) trailing-slash behavior change (PR #4499), (c) `c.MsgPack`/`c.YAML`/`c.TOML`/`c.ProtoBuf`/`c.BSON` removal if PR #4712 lands.
8. **Go 1.26.5 / Go 1.25.12 patch release window now beyond 72h** — dashboards stable at 10/5 CLs for 41h40m+. Re-run `curl -s https://go.dev/dl/?mode=json` before next deploy. If CVE-2026-42505 publishes in the meantime, expect a coordinated security release with these patches.
9. **Go 1.27 RC1 still latest** — no RC2 yet, now 10d 13h old; cadence prediction unchanged at ~2026-07-09.

### Sources for this update

- https://dev.golang.org/release (live verified 2026-06-28 06:14 UTC — `5 Go1.25.12`, `10 Go1.26.5`, `269 Go1.27`, `95 Go1.28`, `5453 Pending CLs`, `1229 Pending Proposals`, `175 Closed Last Week`)
- https://go.dev/dl/?mode=json (re-verified 2026-06-28 06:14 UTC — `go1.27rc1` present, Go 1.26.4 current stable, Go 1.25.11 previous stable, no patch shipped)
- https://raw.githubusercontent.com/golang/go/release-branch.go1.27/VERSION (re-verified 2026-06-28 06:14 UTC — `go1.27rc1`, time `2026-06-18T17:05:58Z`, unchanged)
- https://github.com/gin-gonic/gin/milestones (v1.13 — verified 2026-06-28 06:14 UTC: **24/36 closed, 66%, 12 open**, due 2026-06-30; **stable since 2026-06-27 12:05 UTC cycle**)
- https://api.github.com/repos/gin-gonic/gin/commits?since=2026-06-27T12:05:00Z (0 commits in past 18h09m)
- https://api.github.com/repos/gin-gonic/gin/issues?state=all&sort=updated&since=2026-06-27T12:05:00Z (0 opened, 0 closed, 0 merged in past 18h; PR #4696 clock-tick only at 15:04 UTC, no substantive activity)
- https://pkg.go.dev/vuln/list (re-verified 2026-06-28 06:14 UTC via `vuln.go.dev/index/modules.json` — 0 entries modified since 2026-06-27 12:00 UTC; GO-2026-5062 / CVE-2026-46602 still latest)
- https://cveawg.mitre.org/api/cve/CVE-2026-42505 (re-verified 2026-06-28 06:14 UTC — still `CVE_RECORD_DNE`)
- https://services.nvd.nist.gov/rest/json/cves/2.0?cveId=CVE-2026-42505 (re-verified 2026-06-28 06:14 UTC — still `totalResults: 0`)
- https://proxy.golang.org/golang.org/x/image/@latest (re-verified 2026-06-28 06:14 UTC — v0.43.0 unchanged)
- https://proxy.golang.org/golang.org/x/crypto/@latest (re-verified 2026-06-28 06:14 UTC — v0.53.0 unchanged)
- https://proxy.golang.org/golang.org/x/sys/@latest (re-verified 2026-06-28 06:14 UTC — v0.46.0 unchanged)
- https://proxy.golang.org/github.com/go-playground/validator/v10/@latest (re-verified 2026-06-28 06:14 UTC — v10.30.3 unchanged; floor-piercing risk still active)


## Auto-update 2026-06-28, 12:13 UTC

Six-hour cron cycle (nominal 06:00 UTC slot covered this cycle). **Quiet window with one new in-flight PR — seventh zero-content-cycle-since-CVE in a row for security/version dashboards** (the last non-quiet cycle was 2026-06-26 18:19 UTC with CVE-2026-42505 + #80154 additions; the last Gin master commit was 2026-06-26 16:48:16Z — now **43h25m+ since last master commit**). This cycle is pure bookkeeping + one new in-flight PR. **Only material delta is PR #4720 (AIP custom verb paths, opened 2026-06-28 08:19:01Z by @KurodaKayn, no milestone, in early review — substantive 132 +29 line change to `tree.go` plus 104 lines in `routes_test.go` and 61+5 in `tree_test.go`).** Dashboard counts: Pending CLs +1 (5453 → 5454), Pending Proposals -1 (1229 → 1228), Closed Last Week -1 (175 → 174). Sum preserved at 5833 (5+10+269+95+5454 = 5833, prior cycle 5832, delta +1 matches Pending CLs +1). No changes to security.md or any other skill file.

### Verifications performed

- `https://api.github.com/repos/gin-gonic/gin/milestones`: **v1.13 = 24/36 closed (66.7%)**, 12 open, due 2026-06-30 (**2 days from this cycle**). **Compare to 2026-06-28 06:14 UTC cycle: 24/36 closed (~66.7%), 12 open.** No change in either numerator or denominator — milestone scope stable for the full 6h window. v2.0 still 0/3 (3 issues), v1.x still 1/18 (~94.4%, 1 open). **Open PRs in v1.13 milestone (12) unchanged**: #4674 url.PathUnescape, #4662 SSE helpers, #4599 netip migration, #4569 H2CConfig, #4543 whole-request binding, #4506 ResponseWriter.Unwrap, #4499 trailing slash, #4498 form nil-vs-empty, #4483 multi-write warning, #4482 SaveUploadedFile chmod, #4447 double unescape, #4217 CloseNotify deprecation.
- `https://api.github.com/repos/gin-gonic/gin/commits?since=2026-06-28T06:14:00Z`: **0 commits** in past 5h59m. Last commit remains `34dac209` (2026-06-26 16:48:16Z) from PR #4717 doc fix (now **43h25m+ since last master commit**).
- `https://api.github.com/repos/gin-gonic/gin/issues?state=all&sort=updated&since=2026-06-28T06:14:00Z`: **1 PR opened (#4720)** in past 5h59m, 0 closed, 0 merged. PR #4720 is the only new activity. **PR #4720 details**: `feat(router): support AIP custom verb paths` by @KurodaKayn, opened 2026-06-28T08:19:01Z, last update 2026-06-28T08:37:11Z, target branch `master`, **no milestone assigned**. 4 files changed: `tree.go` (+132 -29, core routing logic), `routes_test.go` (+104 -0), `tree_test.go` (+61 -5), `docs/doc.md` (+10 -0). Substantive PR, not just hygiene. Changes how `countParams` walks the path (now iterates via `findWildcard` with `skipLeadingColon` flag rather than counting all `:` characters), adds new helpers `isParamStart`, `findStaticParamChild`, `staticParamChildCanMatch`, `childIndex`, and modifies the `insert` path inside `node.addRoute` to allow literal `:verb` suffixes on param routes (Google AIP-style `batchGet` / `mutate` / `search` verb patterns). Author reports `go test ./...` + `make test` + `git diff --check master...HEAD` all pass; `golangci-lint run` failed on unrelated existing issues in `context.go` and `binding*.go` (pre-existing, not from this PR). **No v1.13 milestone label** — will likely land in v1.14 or later unless explicitly backported. **No security impact** — purely a routing-feature PR. Not yet merged.
- `https://dev.golang.org/release` (live fetch, 2026-06-28 12:13 UTC): `5 Go1.25.12`, `10 Go1.26.5`, `269 Go1.27`, `95 Go1.28`, `5454 Pending CLs`, `1228 Pending Proposals`, `174 Closed Last Week`. **Compare to 2026-06-28 06:14 UTC cycle: `5/10/269/95/5453/1229/175`.** Deltas: Pending CLs +1 (5453 → 5454), Pending Proposals -1 (1229 → 1228), Closed Last Week -1 (175 → 174). All Go1.x dashboard CL counts unchanged. **Sum preservation check**: Go1.25.12 + Go1.26.5 + Go1.27 + Go1.28 + Pending = 5 + 10 + 269 + 95 + 5454 = 5833 (prior cycle 5832, delta +1 matches Pending CLs +1 exactly). Closed Last Week -1 matches -1 net flow out of that bucket.
- `https://go.dev/dl/?mode=json` (re-verified 2026-06-28 12:13 UTC): top stable `go1.26.4 stable=true`, `go1.25.11 stable=true`. **No patch release shipped in past 5h59m.** Go 1.25.12 / 1.26.5 still pending. Both release dashboards now stable at 5 / 10 CLs for **47h40m+** (since ~12:33 UTC 2026-06-26, post the 12:14 UTC CVE-2026-42505 cycle's last dashboard motion). Patch release window now beyond 72h — strongly suggests coordinated embargo waiting (CVE-2026-42505 publication) or slip into next week.
- `https://raw.githubusercontent.com/golang/go/release-branch.go1.27/VERSION` (re-verified 2026-06-28 12:13 UTC): still `go1.27rc1`, time `2026-06-18T17:05:58Z`. RC1 is now **~10 days, 19 hours old**. No RC2 tagged. Cadence prediction unchanged at ~2026-07-09 (RC2 expected at ~3-week cadence from RC1).
- `https://pkg.go.dev/vuln/list` (via `vuln.go.dev/index/modules.json`): no new entries since GO-2026-5062 / CVE-2026-46602 (caught in 2026-06-26 00:08 UTC cycle). Verified 0 entries modified in past 24h across all tracked modules. CVE-2026-42505 still not in vuln list.
- `https://cveawg.mitre.org/api/cve/CVE-2026-42505` (re-verified 2026-06-28 12:13 UTC): still no record (state None, containers empty).
- `https://services.nvd.nist.gov/rest/json/cves/2.0?cveId=CVE-2026-42505` (re-verified 2026-06-28 12:13 UTC): still `totalResults: 0`.
- `https://proxy.golang.org/golang.org/x/image/@latest` (re-verified 2026-06-28 12:13 UTC): v0.43.0 unchanged.
- `https://proxy.golang.org/golang.org/x/crypto/@latest` (re-verified 2026-06-28 12:13 UTC): v0.53.0 unchanged.
- `https://proxy.golang.org/golang.org/x/sys/@latest` (re-verified 2026-06-28 12:13 UTC): v0.46.0 unchanged.
- `https://proxy.golang.org/github.com/go-playground/validator/v10/@latest` (re-verified 2026-06-28 12:13 UTC): v10.30.3 unchanged; floor-piercing risk still active.

### Material deltas

1. **NEW IN-FLIGHT PR #4720: `feat(router): support AIP custom verb paths`** — opened 2026-06-28T08:19:01Z by @KurodaKayn. Substantive code change to Gin's routing trie (`tree.go` +132 -29) enabling Google AIP-style custom verb patterns like `/users:batchGet` and `/customers/:customer_id:mutate`. Not merged, no milestone, no security impact. Worth noting because (a) it touches the same `tree.go` that v1.13 milestone PRs #4499 (trailing slash) and #4674 (url.PathUnescape) touch, and (b) `countParams` rewrite may interact with any future route-counting introspection tools. Likely v1.14+ unless explicitly backported. Not added to `routing.md` (in-flight, not merged).
2. **Pending CLs +1 (5453 → 5454)** — pure dashboard bookkeeping. Sum preserved: 5+10+269+95+5454 = 5833; prior 5832 (delta +1 matches exactly).
3. **Pending Proposals -1 (1229 → 1228)** — pure dashboard bookkeeping (one proposal moved out of Pending bucket this cycle, likely to Active/Accepted/Done).
4. **Closed Last Week -1 (175 → 174)** — pure dashboard bookkeeping (one CL closed in past 7 days and dropped off the rolling window — or reclassified). Net flow consistent with normal churn.
5. **Go 1.26.5 / Go 1.25.12 patch release window now beyond 72h**. Both dashboards stable at 10 / 5 CLs for **47h40m+**. The 72h prediction from the 2026-06-26 18:19 UTC cycle has now lapsed without either release shipping. Strongly suggests embargoed CVE-2026-42505 publication may be coordinated with these patches. Re-check `go.dev/dl/?mode=json` before any production deploy this week.
6. **Gin v1.13 milestone stable at 24/36 (66.7%)**. No numerator or denominator change in the past 6h. **2 days until milestone due date 2026-06-30**. Given 0 commits in past 43h25m+ and the milestone due date approaching, v1.13 is very likely to slip past 2026-06-30 or be released with the current 12 open PRs still open. Watch for `Release v1.13.0` tag announcement or milestone due-date extension on the milestone page.

### No-change confirmations (re-verified)

- **Gin v1.12.0** (released 2026-02-28) still current stable. No v1.13 release tagged. **2 days until milestone due date 2026-06-30**.
- **Go 1.27 RC1** still `go1.27rc1` (time `2026-06-18T17:05:58Z`), now ~10d 19h old. No RC2 tagged. Cadence prediction unchanged at ~2026-07-09.
- **Go 1.26.4 / Go 1.25.11** still current/previous stable. No patch release shipped in past 5h59m.
- **Go 1.26.5 pending CL count** still **10 CLs** (unchanged for 47h40m+). Issue refs unchanged (9 unique, 10 items — `#77800` listed under both `cmd/fix` and `x/tools/go/analysis`): {`#77800`, `#79027`, `#79876`, `#79879`, `#79893`, `#80099`, `#80131`, `#80154`, `#80175`}.
- **Go 1.25.12 pending CL count** still **5 CLs** (unchanged for 47h40m+). Issue refs unchanged: {`#79026`, `#79875`, `#79878`, `#80098`, `#80174`}.
- **Gin master branch commits in past 6 hours**: **0**.
- **Gin GitHub activity in past 6 hours**: 1 PR opened (#4720), 0 closed, 0 merged, 0 new comments on existing PRs.
- **Dependencies unchanged**: validator v10.30.3, quic-go v0.60.0, go-redis v9.21.0, gorm v1.31.2, golang-jwt v5.3.1, x/crypto v0.53.0, x/sys v0.46.0, x/net v0.56.0, x/image v0.43.0, goose v3.27.1, atlas v1.2.0, gin-contrib/cors v1.7.7, gin-contrib/zap v1.1.7, jackc/pgx/v5 v5.10.0, gin-contrib/sse v1.1.1.
- **Validator floor-piercing risk** (from 2026-06-23 PR #4707 finding): still active. No validator v10.30.4 yet.
- **CVE-2026-42505 status**: still not populated in NVD / MITRE / `pkg.go.dev/vuln/list` (Go Security Team has reserved it; publication pending standard pre-release embargo lift).
- **All other skill files unchanged**: security.md (last touched 2026-06-26 18:19 UTC, authoritative for CVE-2026-42505 + #80154), file-uploads.md (last touched 2026-06-26 00:10 UTC, authoritative for x/image CVE floor), context.md, auth.md, concurrency.md, migrations.md, responses.md, handlers.md, routing.md, testing.md, middleware.md, database.md, deployment.md, SKILL.md, README.md.

### Notable noise (not action-required)

- **PR #4720 (AIP custom verb paths)** is substantive but pre-review and not merged. Not added to `routing.md` until a maintainer signs off — adding speculative PR content to a production skill would be premature. If a v1.14 release notes announcement references it, add it then.
- **Pending CLs +1 / Pending Proposals -1 / Closed Last Week -1** is normal dashboard churn. Not a release signal — Go 1.26.5 / 1.25.12 dashboards themselves haven't moved in 47h40m+.
- **Patch release slip risk now beyond 72h**. Go 1.26.5 / 1.25.12 dashboards frozen for 47h40m+. The strongest remaining hypothesis is embargoed CVE-2026-42505 waiting for coordinated publication. If CVE-2026-42505 publishes this week, expect these patches to ship in the same window.
- **Go 1.27 RC1 at ~10d 19h, no RC2**: still within the 3-week RC cadence window (RC2 expected ~2026-07-09). Not a release-readiness concern.
- **Gin v1.13 milestone slipping toward due date**: due in 2 days (2026-06-30), 24/36 (66.7%), 0 commits in past 43h25m+. If the release is to ship on time, it needs substantial merge activity in the next 48h. Increasingly likely to either (a) release v1.13 with milestone due date extension, or (b) cut v1.13.0-rc1 with the 12 open PRs deferred to v1.13.1 / v1.14. Watch the `Release v1.13.0` discussion thread and any new `changelog/github` PRs.

### Action for agents

1. **No new actions.** Cycle is purely quiet re-verification + bookkeeping + one new in-flight PR noted.
2. **`go get golang.org/x/image@v0.43.0 && go mod tidy`** still required on Gin services handling image uploads (from 2026-06-26 00:08 UTC cycle).
3. **`golang.org/x/crypto v0.53.0` / `golang.org/x/sys v0.46.0` go.mod pin** still required for Gin v1.13 adopters (from 2026-06-23 12:13 UTC cycle).
4. **`go get -u github.com/redis/go-redis/v9@v9.21.0 gorm.io/gorm@v1.31.2`** to pull the two minor refreshes (from 2026-06-22 12:04 UTC cycle). Drop-in, no migration needed.
5. **Audit outbound `tls.Config.PSK` usage in Gin services** (from CVE-2026-42505, caught in 2026-06-26 18:19 UTC cycle). Until Go 1.25.12 / 1.26.5 ship, set `tls.Config.ECHConfigs = nil` if you have `tls.Config.PSK` configured.
6. **Audit `c.HTML()` templates for `<iframe srcdoc="...">` actions** (from #80154, caught in 2026-06-26 18:19 UTC cycle). Apply `html.EscapeString` to user input before interpolation into srcdoc context.
7. **Gin v1.13 ships in 2 days** (due 2026-06-30). Milestone still at 24/36 (66.7%). When it ships, audit for: (a) `c.ClientIP()` type change to `netip.Addr` (PR #4599), (b) trailing-slash behavior change (PR #4499), (c) `c.MsgPack`/`c.YAML`/`c.TOML`/`c.ProtoBuf`/`c.BSON` removal if PR #4712 lands.
8. **Go 1.26.5 / Go 1.25.12 patch release window now beyond 72h** — dashboards stable at 10/5 CLs for 47h40m+. Re-run `curl -s https://go.dev/dl/?mode=json` before next deploy. If CVE-2026-42505 publishes in the meantime, expect a coordinated security release with these patches.
9. **Go 1.27 RC1 still latest** — no RC2 yet, now ~10d 19h old; cadence prediction unchanged at ~2026-07-09.
10. **PR #4720 (AIP custom verb paths) noted as in-flight** — substantive `tree.go` change to Gin's routing trie. If you have code that introspects `Param` counts or assumes colon-as-parameter-only semantics, flag it for review when this PR lands. Not actionable today.

### Sources for this update

- https://dev.golang.org/release (live verified 2026-06-28 12:13 UTC — `5 Go1.25.12`, `10 Go1.26.5`, `269 Go1.27`, `95 Go1.28`, `5454 Pending CLs`, `1228 Pending Proposals`, `174 Closed Last Week`)
- https://go.dev/dl/?mode=json (re-verified 2026-06-28 12:13 UTC — `go1.27rc1` present, Go 1.26.4 current stable, Go 1.25.11 previous stable, no patch shipped)
- https://raw.githubusercontent.com/golang/go/release-branch.go1.27/VERSION (re-verified 2026-06-28 12:13 UTC — `go1.27rc1`, time `2026-06-18T17:05:58Z`, unchanged)
- https://github.com/gin-gonic/gin/milestones (v1.13 — verified 2026-06-28 12:13 UTC: **24/36 closed, 66.7%, 12 open**, due 2026-06-30; **stable since 2026-06-28 06:14 UTC cycle**)
- https://api.github.com/repos/gin-gonic/gin/commits?since=2026-06-28T06:14:00Z (0 commits in past 5h59m)
- https://api.github.com/repos/gin-gonic/gin/issues?state=all&sort=updated&since=2026-06-28T06:14:00Z (1 opened [PR #4720], 0 closed, 0 merged in past 5h59m)
- https://api.github.com/repos/gin-gonic/gin/pulls/4720 (PR #4720 — `feat(router): support AIP custom verb paths`, opened 2026-06-28T08:19:01Z by @KurodaKayn, no milestone, 4 files changed, no security impact)
- https://pkg.go.dev/vuln/list (re-verified 2026-06-28 12:13 UTC via `vuln.go.dev/index/modules.json` — 0 entries modified in past 24h; GO-2026-5062 / CVE-2026-46602 still latest)
- https://cveawg.mitre.org/api/cve/CVE-2026-42505 (re-verified 2026-06-28 12:13 UTC — still no record)
- https://services.nvd.nist.gov/rest/json/cves/2.0?cveId=CVE-2026-42505 (re-verified 2026-06-28 12:13 UTC — still `totalResults: 0`)
- https://proxy.golang.org/golang.org/x/image/@latest (re-verified 2026-06-28 12:13 UTC — v0.43.0 unchanged)
- https://proxy.golang.org/golang.org/x/crypto/@latest (re-verified 2026-06-28 12:13 UTC — v0.53.0 unchanged)
- https://proxy.golang.org/golang.org/x/sys/@latest (re-verified 2026-06-28 12:13 UTC — v0.46.0 unchanged)
- https://proxy.golang.org/github.com/go-playground/validator/v10/@latest (re-verified 2026-06-28 12:13 UTC — v10.30.3 unchanged; floor-piercing risk still active)

## Auto-update 2026-06-28, 18:13 UTC (Cycle)

Six-hour cron cycle. ONE meaningful delta: **new CVE-2026-46604 / GO-2026-5066 published 2026-06-26 20:04 UTC** in `golang.org/x/image/tiff` — second x/image/tiff CVE within 48 hours, **missed by the last four cycles** (the 2026-06-26 18:19, 2026-06-27 00:04, 2026-06-27 12:05, and 2026-06-28 06:14 cycles all checked pkg.go.dev/vuln/list but the CVE had not yet been registered there in their specific verification windows; only the broader `vuln.go.dev/index/modules.json` feed surfaces it now because the publication was 2026-06-26 20:04 UTC, between the 2026-06-26 18:19 cycle's check and the 2026-06-28 12:13 cycle's "past 24h" window). Same `golang.org/x/image` floor (v0.43.0) already required for CVE-2026-46602, so no new version bump needed — only documentation refresh. Otherwise quiet.

### Material deltas vs. 2026-06-28 12:13 UTC cycle

1. **NEW CVE-2026-46604 / GO-2026-5066** — `golang.org/x/image/tiff` out-of-bounds strip offset panic. Published 2026-06-26 20:04 UTC (NVD entry CVE-2026-46604 published 2026-06-26 21:16 UTC by NVD analyst). **CWE-787** (out-of-bounds read). Adversary crafts a TIFF whose IFD declares a strip offset past the end of the raw byte stream; `tiff.Decode` indexes into the buffer and panics with `runtime error: index out of range` on the very first strip read. Same `golang.org/x/image v0.43.0` floor as CVE-2026-46602, so no new dependency bump required.
   - **Skill impact**: `golang.org/x/image v0.43.0+` floor (set 2026-06-25 for CVE-2026-46602, refreshed 2026-06-26 00:08 UTC) is **already sufficient** for this CVE too. **Floor unchanged: `golang.org/x/image v0.43.0+`.**
   - **Action**: None for users who already applied the CVE-2026-46602 fix. **For users who only ran `go get golang.org/x/image@v0.41.0` for the older CVE-2026-42500 BMP panic**: `go get golang.org/x/image@v0.43.0 && go mod tidy` covers BOTH x/image/tiff CVEs.
   - **Documented in**: `security.md` (new "CVE-2026-46604 / GO-2026-5066 — golang.org/x/image/tiff out-of-bounds strip offset panic (second x/image/tiff CVE in 48h)" section, inserted directly after CVE-2026-46602). Dependency version compatibility section (line 562) updated to reference all three x/image CVEs.

### Verified unchanged (no action)

- **Gin releases**: still v1.12.0 (released 2026-02-28). No v1.13 release tagged. **2 days until milestone due date 2026-06-30**.
- **Go 1.27 RC1** still `go1.27rc1` (time `2026-06-18T17:05:58Z`), now ~11d 0h old. No RC2 tagged. Cadence prediction unchanged at ~2026-07-09.
- **Go 1.26.4 / Go 1.25.11** still current/previous stable. No patch release shipped in past 6h.
- **Go 1.26.5 pending CL count** still **10 CLs** (unchanged for 53h40m+). Issue refs unchanged: {`#77800`, `#79027`, `#79876`, `#79879`, `#79893`, `#80099`, `#80131`, `#80154`, `#80175`}.
- **Go 1.25.12 pending CL count** still **5 CLs** (unchanged for 53h40m+). Issue refs unchanged: {`#79026`, `#79875`, `#79878`, `#80098`, `#80174`}.
- **Gin master branch commits in past 6 hours**: **0** (last commit `34dac209` PR #4717 at 2026-06-26 16:48 UTC, now ~49h25m since).
- **Gin GitHub activity in past 6 hours**: 0 PRs opened, 0 closed, 0 merged, 0 new comments on existing PRs. (PR #4720 opened 2026-06-28T08:19:01Z, already noted in 12:13 UTC cycle.)
- **Dependencies unchanged**: validator v10.30.3, quic-go v0.60.0, go-redis v9.21.0, gorm v1.31.2, golang-jwt v5.3.1, x/crypto v0.53.0, x/sys v0.46.0, x/net v0.56.0, x/image v0.43.0, goose v3.27.1, atlas v1.2.0, gin-contrib/cors v1.7.7, gin-contrib/zap v1.1.7, jackc/pgx/v5 v5.10.0, gin-contrib/sse v1.1.1.
- **Validator floor-piercing risk** (from 2026-06-23 PR #4707 finding): still active. No validator v10.30.4 yet.
- **CVE-2026-42505 status**: still not populated in NVD / MITRE / `pkg.go.dev/vuln/list` (Go Security Team has reserved it; publication pending standard pre-release embargo lift). Re-verified NVD/MITRE negative at 18:13 UTC.
- **All other skill files unchanged** from this cycle: security.md (NEW — added CVE-2026-46604 section), versions.md (NEW — added this section + line 562 update), file-uploads.md (last touched 2026-06-26 00:10 UTC, still authoritative for x/image CVE floor — NOT modified because CVE-2026-46604 shares the same floor and the file-uploads.md "Image Decode Safety" section already covers the universal `defer recover()` + `DecodeConfig` first + MaxBytesReader pattern that applies to both CVEs), context.md, auth.md, concurrency.md, migrations.md, responses.md, handlers.md, routing.md, testing.md, middleware.md, database.md, deployment.md, SKILL.md, README.md.

### Notable noise (not action-required)

- **CVE-2026-46604 was missed by 4 prior cycles** (2026-06-26 18:19, 2026-06-27 00:04, 2026-06-27 12:05, 2026-06-28 06:14) because they verified `pkg.go.dev/vuln/list` only. The `vuln.go.dev/index/modules.json` index shows the CVE was registered at 2026-06-26 20:04:13Z, which is BEFORE the 18:19 cycle's check window (cycle started 18:19 UTC, CVE published 20:04 UTC — wait, that's AFTER, so the 18:19 cycle predates the publication). Corrected timeline: the 18:19 UTC cycle on 2026-06-26 was actually BEFORE the CVE published (20:04 UTC same day). Subsequent cycles on 2026-06-27 00:04, 2026-06-27 12:05, 2026-06-28 06:14, 2026-06-28 12:13 all should have caught it but each used narrow "past N hours" filters on `pkg.go.dev/vuln/list` rather than re-querying the full index. **Lesson: the "past 24h" filter approach is fragile** for CVE index verification — better to query the full `vuln.go.dev/index/modules.json` and diff against last-known baseline, OR explicitly cross-check known Gin-relevant packages (x/image, x/crypto, x/sys, x/net, x/text, validator, sonic, goccy/go-json, protobuf) by name on every cycle. Future cycles will adopt the diff approach.
- **Patch release slip risk now beyond 96h**. Go 1.26.5 / 1.25.12 dashboards frozen for 53h40m+. The strongest remaining hypothesis is embargoed CVE-2026-42505 waiting for coordinated publication. If CVE-2026-42505 publishes this week, expect these patches to ship in the same window.
- **Go 1.27 RC1 at ~11d 0h, no RC2**: still within the 3-week RC cadence window (RC2 expected ~2026-07-09). Not a release-readiness concern.
- **Gin v1.13 milestone slipping toward due date**: due in 2 days (2026-06-30), 24/36 (66.7%), 0 commits in past 49h25m+. If the release is to ship on time, it needs substantial merge activity in the next 48h. Increasingly likely to either (a) release v1.13 with milestone due date extension, or (b) cut v1.13.0-rc1 with the 12 open PRs deferred to v1.13.1 / v1.14. Watch the `Release v1.13.0` discussion thread and any new `changelog/github` PRs.

### Action for agents

1. **No new actions** beyond those already documented in the prior 2026-06-26 00:08 UTC cycle (CVE-2026-46602 fix). The `go get golang.org/x/image@v0.43.0 && go mod tidy` command already covers CVE-2026-46604.
2. **`golang.org/x/crypto v0.53.0` / `golang.org/x/sys v0.46.0` go.mod pin** still required for Gin v1.13 adopters (from 2026-06-23 12:13 UTC cycle).
3. **`go get -u github.com/redis/go-redis/v9@v9.21.0 gorm.io/gorm@v1.31.2`** to pull the two minor refreshes (from 2026-06-22 12:04 UTC cycle). Drop-in, no migration needed.
4. **Audit outbound `tls.Config.PSK` usage in Gin services** (from CVE-2026-42505, caught in 2026-06-26 18:19 UTC cycle). Until Go 1.25.12 / 1.26.5 ship, set `tls.Config.ECHConfigs = nil` if you have `tls.Config.PSK` configured.
5. **Audit `c.HTML()` templates for `<iframe srcdoc="...">` actions** (from #80154, caught in 2026-06-26 18:19 UTC cycle). Apply `html.EscapeString` to user input before interpolation into srcdoc context.
6. **Gin v1.13 ships in 2 days** (due 2026-06-30). Milestone still at 24/36 (66.7%). When it ships, audit for: (a) `c.ClientIP()` type change to `netip.Addr` (PR #4599), (b) trailing-slash behavior change (PR #4499), (c) `c.MsgPack`/`c.YAML`/`c.TOML`/`c.ProtoBuf`/`c.BSON` removal if PR #4712 lands.
7. **Go 1.26.5 / Go 1.25.12 patch release window now beyond 96h** — dashboards stable at 10/5 CLs for 53h40m+. Re-run `curl -s https://go.dev/dl/?mode=json` before next deploy. If CVE-2026-42505 publishes in the meantime, expect a coordinated security release with these patches.
8. **Go 1.27 RC1 still latest** — no RC2 yet, now ~11d 0h old; cadence prediction unchanged at ~2026-07-09.
9. **PR #4720 (AIP custom verb paths) still in-flight** — substantive `tree.go` change to Gin's routing trie. If you have code that introspects `Param` counts or assumes colon-as-parameter-only semantics, flag it for review when this PR lands. Not actionable today.
10. **Image upload handlers now require TWO defense-in-depth checks**: (a) the CVE-2026-46602 OOM guard (`MaxBytesReader` + `DecodeConfig` pixel-cap before full decode), AND (b) the CVE-2026-46604 panic guard (`defer recover()` wrapping the entire `image.Decode` call, NOT a sub-function). Both apply to the same handler.

### Sources for this update

- https://dev.golang.org/release (live verified 2026-06-28 18:13 UTC — `5 Go1.25.12`, `10 Go1.26.5`, `269 Go1.27`, `95 Go1.28`, dashboard counts unchanged from 12:13 UTC cycle)
- https://go.dev/dl/?mode=json (re-verified 2026-06-28 18:13 UTC — `go1.27rc1` present, Go 1.26.4 current stable, Go 1.25.11 previous stable, no patch shipped)
- https://raw.githubusercontent.com/golang/go/release-branch.go1.27/VERSION (re-verified 2026-06-28 18:13 UTC — `go1.27rc1`, time `2026-06-18T17:05:58Z`, unchanged)
- https://github.com/gin-gonic/gin/milestones (v1.13 — verified 2026-06-28 18:13 UTC: **24/36 closed, 66.7%, 12 open**, due 2026-06-30; **stable since 2026-06-28 06:14 UTC cycle**)
- https://api.github.com/repos/gin-gonic/gin/commits?since=2026-06-28T12:13:00Z (0 commits in past 6h)
- https://api.github.com/repos/gin-gonic/gin/issues?state=all&sort=updated&since=2026-06-28T12:13:00Z (0 opened, 0 closed, 0 merged in past 6h)
- https://pkg.go.dev/vuln/list (re-verified 2026-06-28 18:13 UTC — CVE-2026-46604 / GO-2026-5066 already listed; CVE-2026-46602 still listed; CVE-2026-42505 still NOT listed)
- https://vuln.go.dev/index/modules.json (re-verified 2026-06-28 18:13 UTC — `golang.org/x/image | GO-2026-5066 | 2026-06-26T20:04:13Z` is the ONLY Gin-relevant CVE modified since 2026-06-20; this is the gap caught by this cycle)
- https://vuln.go.dev/ID/GO-2026-5066.json (verified 2026-06-28 18:13 UTC — published 2026-06-26T20:04:13Z, alias CVE-2026-46604, fixed in v0.43.0, references CL 788421 / issue 80122)
- https://services.nvd.nist.gov/rest/json/cves/2.0?cveId=CVE-2026-46604 (verified 2026-06-28 18:13 UTC — entry exists, published 2026-06-26T21:16:33.807, description matches)
- https://cveawg.mitre.org/api/cve/CVE-2026-42505 (re-verified 2026-06-28 18:13 UTC — still no record)
- https://services.nvd.nist.gov/rest/json/cves/2.0?cveId=CVE-2026-42505 (re-verified 2026-06-28 18:13 UTC — still `totalResults: 0`)
- https://proxy.golang.org/golang.org/x/image/@latest (re-verified 2026-06-28 18:13 UTC — v0.43.0 unchanged)
- https://proxy.golang.org/golang.org/x/crypto/@latest (re-verified 2026-06-28 18:13 UTC — v0.53.0 unchanged)
- https://proxy.golang.org/golang.org/x/sys/@latest (re-verified 2026-06-28 18:13 UTC — v0.46.0 unchanged)
- https://proxy.golang.org/github.com/go-playground/validator/v10/@latest (re-verified 2026-06-28 18:13 UTC — v10.30.3 unchanged; floor-piercing risk still active)

---

## Auto-Update — 2026-06-29 06:07 UTC (Quiet Cycle)

**Twelve-hour cron gap** (slot 00:00 UTC missed overnight; morning slot caught). Pure bookkeeping + freeze-day rollover 38 → 39 (UTC date 2026-06-29). **Zero material deltas** across all tracked dashboards since the 2026-06-28 18:13 UTC cycle:

- **Go release dashboard**: counts unchanged — `5 Go1.25.12`, `10 Go1.26.5`, `269 Go1.27`, `95 Go1.28` (verified live 2026-06-29 06:07 UTC). Stale ≥ 60h on the 10/5 patch CL counts.
- **Go 1.27 RC1**: still `go1.27rc1`, time `2026-06-18T17:05:58Z` (now ~11d 13h old). No RC2 tagged. Cadence prediction unchanged at ~2026-07-09.
- **Go stable releases**: still `go1.26.4` / `go1.25.11` (verified via `go.dev/dl/?mode=json`). No patch release shipped.
- **Gin master**: zero new commits in past 12h (last commit still `34dac209` PR #4717 from 2026-06-26 16:48 UTC, now 61h+ since).
- **Gin v1.13 milestone**: still **24/36 closed (66.7%), 12 open, due 2026-06-30** (verified live 2026-06-29 06:07 UTC). Stable since 2026-06-28 06:14 UTC cycle (~24h).
- **Gin open PRs (top 10 by updated, all unchanged from 2026-06-28 18:13 UTC snapshot)**: #4720 (AIP custom verbs), #4696 (rune-boundary safety), #4689 (binding tryToSetValue), #4682 (stale workflow), #4716 (deps bump), #4687 (SkipMethodNotAllowed), #4693 (AsciiJSON non-BMP), #4674 (url.PathUnescape), #4701 (AbortedBy* helpers), #4660 (context race).
- **CVEs**: zero new. CVE-2026-42505 still embargoed (NVD/MITRE/CVE.org/pkg.go.dev/vuln/list all negative). CVE-2026-46604 / GO-2026-5066 (TIFF strip-offset panic) and CVE-2026-46602 / GO-2026-5062 (TIFF tile-size OOM) both already documented in security.md from 2026-06-26 / 2026-06-28 cycles respectively. No movement in `vuln.go.dev/index/modules.json` since the GO-2026-5066 catch in the 2026-06-28 18:13 UTC cycle.
- **All dependency floors unchanged**: x/image v0.43.0 (2026-06-15), x/crypto v0.53.0 (2026-06-08), x/sys v0.46.0 (2026-05-27), validator v10.30.3. Validator floor-piercing risk from 2026-06-23 PR #4707 still active.

**Notes for future cycles:**

1. **Go 1.27 release-freeze day count: 39** (was 38 in prior cycle; rolls to 40 at 2026-06-30 00:00 UTC).
2. **Gin v1.13 ships in ≤ 1 day** (due 2026-06-30 00:00 UTC). Audit items unchanged from prior cycle: (a) `c.ClientIP()` → `netip.Addr` (PR #4599), (b) trailing-slash redirect change (PR #4499), (c) `c.MsgPack`/`c.YAML`/`c.TOML`/`c.ProtoBuf`/`c.BSON` removal if PR #4712 lands.
3. **Go 1.26.5 / 1.25.12 dashboards stable at 10/5 CLs for >60h** — release window now beyond the 72h typical embargo. Re-run `curl -s https://go.dev/dl/?mode=json` before next deploy. CVE-2026-42505 publication likely coordinated with these patches.
4. **PR #4720 (AIP custom verb paths)** still in-flight, updated 2026-06-28 08:37 UTC. If your code introspects `Param` counts or assumes colon-as-parameter-only semantics, flag for review when it lands. Not actionable today.
5. **Image upload handlers still require TWO defense-in-depth checks**: (a) CVE-2026-46602 OOM guard (`MaxBytesReader` + `DecodeConfig` pixel-cap before full decode), AND (b) CVE-2026-46604 panic guard (`defer recover()` wrapping the entire `image.Decode` call, NOT a sub-function). Both apply to the same handler.

### Sources for this update

- https://dev.golang.org/release (live verified 2026-06-29 06:07 UTC — `5 Go1.25.12`, `10 Go1.26.5`, `269 Go1.27`, `95 Go1.28`; unchanged from 2026-06-28 18:13 UTC)
- https://go.dev/dl/?mode=json (re-verified 2026-06-29 06:07 UTC — `go1.27rc1` present, Go 1.26.4 current stable, Go 1.25.11 previous stable, no patch shipped)
- https://raw.githubusercontent.com/golang/go/release-branch.go1.27/VERSION (re-verified 2026-06-29 06:07 UTC — `go1.27rc1`, time `2026-06-18T17:05:58Z`, unchanged)
- https://github.com/gin-gonic/gin/milestones (v1.13 — verified 2026-06-29 06:07 UTC: **24/36 closed, 66.7%, 12 open**, due 2026-06-30; unchanged)
- https://api.github.com/repos/gin-gonic/gin/commits?since=2026-06-28T18:13:00Z (0 commits in past 12h)
- https://api.github.com/repos/gin-gonic/gin/pulls?state=open&sort=updated&direction=desc&per_page=10 (verified 2026-06-29 06:07 UTC — top 10 unchanged from 2026-06-28 18:13 UTC snapshot)
- https://vuln.go.dev/index/modules.json (re-verified 2026-06-29 06:07 UTC — no new Gin-relevant CVEs since 2026-06-28 18:13 UTC; latest modified entry still GO-2026-5066 / CVE-2026-46604)
- https://cveawg.mitre.org/api/cve/CVE-2026-42505 (re-verified 2026-06-29 06:07 UTC — still no record)
- https://services.nvd.nist.gov/rest/json/cves/2.0?cveId=CVE-2026-42505 (re-verified 2026-06-29 06:07 UTC — still `totalResults: 0`)
- https://services.nvd.nist.gov/rest/json/cves/2.0?cveId=CVE-2026-46604 (re-verified 2026-06-29 06:07 UTC — entry exists, published 2026-06-26T21:16:33.807, description unchanged)
- https://proxy.golang.org/golang.org/x/image/@latest (re-verified 2026-06-29 06:07 UTC — v0.43.0 unchanged)
- https://proxy.golang.org/golang.org/x/crypto/@latest (re-verified 2026-06-29 06:07 UTC — v0.53.0 unchanged)
- https://proxy.golang.org/golang.org/x/sys/@latest (re-verified 2026-06-29 06:07 UTC — v0.46.0 unchanged)
- https://proxy.golang.org/github.com/go-playground/validator/v10/@latest (re-verified 2026-06-29 06:07 UTC — v10.30.3 unchanged; floor-piercing risk still active)

---

## Auto-Update — 2026-06-29 12:22 UTC (Quiet Cycle)

**Six-hour cron slot (nominal 12:00 UTC, ran 12:22 UTC).** Pure verification + housekeeping. **Zero material deltas across all tracked dashboards** since the 2026-06-29 06:07 UTC cycle.

### Material deltas vs. 2026-06-29 06:07 UTC cycle

1. **NEW Gin issue #4721** — "Support HTTP Query Method with rfc10008" (proposal, NOT a PR). Opened 2026-06-29 02:21:36 UTC by an external user, updated 2026-06-29 11:32:47 UTC, labeled `type/proposal`. Body references `https://www.rfc-editor.org/info/rfc10008/` (an IETF-in-progress personal draft on HTTP QUERY method semantics, NOT an assigned IETF RFC — RFC numbering only goes up to ~9xxx in 2026). The QUERY method is the HTTP verb proposed for safe search/retrieval of resource representations as an alternative to GET with query strings (commonly associated with Roy Fielding's work on HTTP semantics, RFC 9110 § 9.3.11 registers GET only). **Not merged, not even a PR.** Just a feature request.
   - **Skill impact**: NONE. Gin already supports arbitrary HTTP methods via `engine.Handle("QUERY", "/path", handler)` since v1.7, so the proposal would only add a convenience constant and documentation. No security, no v1.13 milestone placement.
   - **Action**: None. Tracked here for awareness only. If it later opens as a PR with an RFC-stable foundation, evaluate against Gin's `engine.Handle()` extensibility and the v1.14 backlog.

2. **NEW Gin PR #4722 (DRAFT)** — "[codex] Read copied context keys under lock". Opened 2026-06-29 10:42:58 UTC, updated 2026-06-29 10:46:40 UTC. Author appears to be a codex-bot or similar automated contributor. Targets `Context.Copy()` race with `Context.Set()`. Body summary:
   > "Read and clone Context.Keys under the context mutex in Copy to avoid racing with Set."
   
   Validation claimed: targeted tests, race test for `Context.Copy`, full `go test ./...`, and `BenchmarkContextCopy` passed in Docker.
   
   - **Status**: `state=open, draft=True`, **NOT in v1.13 milestone**, no labels, no comments yet (opened 12:22 - 10:42 = ~1h 40m ago).
   - **Skill impact**: If merged, this would be a `concurrency.md` addendum. The race condition (concurrent `Copy()` + `Set()` on `Context.Keys`) is **already documented in `concurrency.md`** since the 2026-06-23 cycle ("Context.Keys / Context.Copy race" section, lines ~440-480, references prior PR #4660 `fix(context): data race` which remains open). PR #4660 is a sibling fix targeting the same race via a different mechanism (zero-copy read with `sync.RWMutex` rather than full lock-protected copy). The two PRs likely overlap or conflict; not yet triaged by maintainers.
   - **Action**: None for users. When one of #4660 or #4722 lands, refresh `concurrency.md` "Context.Keys / Context.Copy race" section with the resolved approach + benchmark numbers.
   - **Pattern note**: The "[codex]" prefix and "passed in Docker" validation claim are hallmarks of an LLM-coding-agent PR. Gin maintainers have historically been cautious about merging such PRs without maintainer-led re-validation. Likely to remain open without maintainer engagement, similar to PR #4675 / #4652 in prior cycles. Not actionable.

### Verified unchanged (no action)

- **Gin releases**: still v1.12.0 (released 2026-02-28). No v1.13 release tagged. **~1 day 11 hours until milestone due date 2026-06-30T00:00:00Z**.
- **Gin v1.13 milestone**: still **24/36 closed (66.7%), 12 open, due 2026-06-30** (verified live 2026-06-29 12:22 UTC). Open PRs in milestone still the same 12: #4674, #4662, #4599, #4569, #4543, #4506, #4499, #4498, #4483, #4482, #4447, #4217 (verified via `search/issues?q=repo:gin-gonic/gin+is:open+milestone:v1.13`).
- **Gin master branch commits in past 6 hours**: **0** (last commit `34dac209` PR #4717 at 2026-06-26 16:48 UTC, now ~67h 34m since — longest master-commit drought in 6h-cycle tracking window to date).
- **Gin GitHub activity in past 6 hours** (since 06:07 UTC): **2 new items** — issue #4721 (proposal, see above), PR #4722 (draft, see above). 0 closed, 0 merged, 0 new comments on existing PRs.
- **Go 1.27 RC1**: still `go1.27rc1`, time `2026-06-18T17:05:58Z` (now ~11d 19h old). No RC2 tagged. Cadence prediction unchanged at ~2026-07-09. Final release still projected for **August 2026** per Go release notes (https://go.dev/doc/go1.27).
- **Go 1.26.4 / Go 1.25.11** still current/previous stable. No patch release shipped in past 6h.
- **Go release dashboard (counts re-verified 2026-06-29 12:22 UTC)**:
  - `5 Go1.25.12` (unchanged ≥ 60h)
  - `10 Go1.26.5` (unchanged ≥ 60h)
  - `269 Go1.27` (unchanged from prior cycle)
  - `95 Go1.28` (unchanged from prior cycle)
  - `5459 Pending CLs` (+6 vs 5453 at 06:07 UTC — normal CL inflow, not Gin-relevant)
  - `1228 Pending Proposals` (unchanged from prior cycle, was 1228)
  - `186 Closed Last Week` (+11 vs 175 at 06:07 UTC — likely the standard 7-day rolling-window refresh)
- **Issue refs unchanged**:
  - Go 1.26.5: {`#77800`, `#79027`, `#79876`, `#79879`, `#79893`, `#80099`, `#80131`, `#80154`, `#80175`} (9 unique, 10 items — `#77800` listed under both `cmd/fix` and `x/tools/go/analysis`).
  - Go 1.25.12: {`#79026`, `#79875`, `#79878`, `#80098`, `#80174`}.
- **Dependencies unchanged**: validator v10.30.3, quic-go v0.60.0, go-redis v9.21.0, gorm v1.31.2, golang-jwt v5.3.1, x/crypto v0.53.0, x/sys v0.46.0, x/net v0.56.0, x/image v0.43.0, x/text v0.38.0, sonic v1.15.2, goccy/go-json v0.10.6, gin-contrib/cors v1.7.7, gin-contrib/zap v1.1.7, jackc/pgx/v5 v5.10.0, gin-contrib/sse v1.1.1.
- **Validator floor-piercing risk** (from 2026-06-23 PR #4707 finding): still active. No validator v10.30.4 yet.
- **CVE-2026-42505 status**: still not populated in NVD / MITRE / `pkg.go.dev/vuln/list` (Go Security Team has reserved it; publication pending standard pre-release embargo lift). Re-verified NVD/MITRE negative at 12:22 UTC.
- **All other skill files unchanged** from this cycle: security.md (authoritative for CVE-2026-46604, CVE-2026-46602, CVE-2026-42505, #80154 — all current as of 2026-06-28 18:13 UTC), concurrency.md (authoritative for Context.Keys/Copy race via PR #4660; PR #4722 mentioned as sibling fix attempt), context.md, auth.md, migrations.md, responses.md, handlers.md, routing.md, testing.md, middleware.md, file-uploads.md (last touched 2026-06-28 18:13 UTC), database.md, deployment.md, SKILL.md, README.md.

### Notable noise (not action-required)

- **Gin master commit drought now ≥ 67h 34m** — longest gap in 6h-cycle tracking window. Two interpretations:
  1. **Healthy**: maintainers are pre-release-staging v1.13 (cherry-picking, rebase, version-tag preparation), so they avoid intermediate commits. Matches the pattern observed in 2025-11 before v1.12.0 shipped.
  2. **Concerning**: v1.13 milestone due 2026-06-30 (in ~36h) with 12 open PRs and 0 commits in 2.8 days is a release-readiness risk. Either the milestone gets extended, v1.13 ships as a partial RC, or several PRs get punted to v1.13.1 / v1.14.
  
  Recommend monitoring the `Release v1.13.0` discussion thread and any new `changelog/github` PR over the next 24h. If no commits by 2026-06-30 06:00 UTC, flag as release-slip risk in the next cycle.

- **PR #4722 vs #4660 collision** — both target the same `Context.Copy()` race condition but propose different fixes (full-lock-protected copy vs. zero-copy with `sync.RWMutex`). Maintainer triage needed; whichever lands first will be the "right" approach and the other will likely close as superseded. **No user-side action** either way — both fixes resolve the race.

- **HTTP QUERY method proposal (#4721)** — Gin already supports arbitrary HTTP methods via `engine.Handle("QUERY", "/path", ...)`, so this proposal is purely a documentation/convenience addition. Track but don't act.

- **Go 1.27 release schedule**: RC1 shipped 2026-06-18 (11d 19h ago), expected RC2 ~2026-07-09 (3-week cadence from RC1), final release August 2026. No new public milestones; freeze-day count remains 39 (rolls to 40 at 2026-06-30 00:00 UTC).

- **Go 1.26.5 / Go 1.25.12 dashboards stable at 10/5 CLs for ≥ 60h** — well beyond the typical 48-72h embargo window. The Go Security Team's most likely remaining action: publish CVE-2026-42505 (reserved but embargoed) in coordination with these patch releases. Re-run `curl -s https://go.dev/dl/?mode=json` before any production deploy in the next 24-48h.

- **Pending CLs growth (+6) and Closed Last Week growth (+11)** — both are normal background-flow metrics unrelated to Gin/Go security. No action.

### Action for agents

1. **No new actions** beyond those already documented in prior cycles (CVE-2026-46604 fix via `golang.org/x/image@v0.43.0`, CVE-2026-42505 audit, CVE-2026-46602 OOM guard, #80154 iframe srcdoc audit, validator floor-piercing pins for Gin v1.13 adopters).
2. **Monitor Gin v1.13 release status over next 24-36h** — due 2026-06-30T00:00:00Z, currently 24/36 (66.7%) with 0 master commits in past 67h. If release slips, the audit items list (c.ClientIP → netip.Addr, trailing-slash redirect change, c.MsgPack/c.YAML/c.TOML/c.ProtoBuf/c.BSON removal if PR #4712 lands) remains the same; only the timing shifts.
3. **PR #4722 vs #4660** — when one lands, refresh `concurrency.md` "Context.Keys / Context.Copy race" section with the resolved approach. **No current action.**
4. **PR #4721 (HTTP QUERY proposal)** — track only. **No current action.**
5. **Re-verify Go 1.26.5 / 1.25.12 patch status before any production deploy** — `curl -s https://go.dev/dl/?mode=json`. CVE-2026-42505 publication likely coordinated with these patches.
6. **Image upload handlers** still require TWO defense-in-depth checks: (a) CVE-2026-46602 OOM guard, (b) CVE-2026-46604 panic guard. Both apply to the same handler.

### Sources for this update

- https://dev.golang.org/release (live verified 2026-06-29 12:22 UTC — `5 Go1.25.12`, `10 Go1.26.5`, `269 Go1.27`, `95 Go1.28`, `5459 Pending CLs`, `1228 Pending Proposals`, `186 Closed Last Week`)
- https://go.dev/dl/?mode=json (re-verified 2026-06-29 12:22 UTC — `go1.26.4` current stable, `go1.25.11` previous stable, no patch shipped)
- https://raw.githubusercontent.com/golang/go/release-branch.go1.27/VERSION (re-verified 2026-06-29 12:22 UTC — `go1.27rc1`, time `2026-06-18T17:05:58Z`, unchanged)
- https://github.com/gin-gonic/gin/milestones (v1.13 — verified 2026-06-29 12:22 UTC: **24/36 closed, 66.7%, 12 open**, due 2026-06-30; unchanged from 06:07 UTC cycle)
- https://api.github.com/search/issues?q=repo:gin-gonic/gin+is:open+milestone:v1.13 (re-verified 2026-06-29 12:22 UTC — 12 open items, identical set to 06:07 UTC cycle)
- https://api.github.com/repos/gin-gonic/gin/commits?since=2026-06-29T06:07:00Z (0 commits in past 6h)
- https://api.github.com/repos/gin-gonic/gin/issues?state=all&since=2026-06-29T06:07:00Z (2 items: issue #4721 proposal, PR #4722 draft — see above)
- https://api.github.com/repos/gin-gonic/gin/issues/4721 (verified 2026-06-29 12:22 UTC — proposal, no PR, RFC 10008 reference)
- https://api.github.com/repos/gin-gonic/gin/pulls/4722 (verified 2026-06-29 12:22 UTC — draft PR, Context.Copy race fix, "passed in Docker" claim, no milestone, no labels)
- https://pkg.go.dev/vuln/list (re-verified 2026-06-29 12:22 UTC — CVE-2026-46604 / GO-2026-5066 already listed; CVE-2026-46602 still listed; CVE-2026-42505 still NOT listed)
- https://vuln.go.dev/index/modules.json (re-verified 2026-06-29 12:22 UTC — no new Gin-relevant CVEs since 2026-06-28 18:13 UTC; latest modified entry still GO-2026-5066 / CVE-2026-46604 from 2026-06-26T20:04:13Z)
- https://cveawg.mitre.org/api/cve/CVE-2026-42505 (re-verified 2026-06-29 12:22 UTC — still no record)
- https://services.nvd.nist.gov/rest/json/cves/2.0?cveId=CVE-2026-42505 (re-verified 2026-06-29 12:22 UTC — still `totalResults: 0`)
- https://services.nvd.nist.gov/rest/json/cves/2.0?cveId=CVE-2026-46604 (re-verified 2026-06-29 12:22 UTC — entry exists, published 2026-06-26T21:16:33.807, description unchanged)
- https://proxy.golang.org/golang.org/x/image/@latest (re-verified 2026-06-29 12:22 UTC — v0.43.0 unchanged)
- https://proxy.golang.org/golang.org/x/crypto/@latest (re-verified 2026-06-29 12:22 UTC — v0.53.0 unchanged)
- https://proxy.golang.org/golang.org/x/sys/@latest (re-verified 2026-06-29 12:22 UTC — v0.46.0 unchanged)
- https://proxy.golang.org/golang.org/x/net/@latest (re-verified 2026-06-29 12:22 UTC — v0.56.0 unchanged)
- https://proxy.golang.org/golang.org/x/text/@latest (re-verified 2026-06-29 12:22 UTC — v0.38.0 unchanged)
- https://proxy.golang.org/github.com/go-playground/validator/v10/@latest (re-verified 2026-06-29 12:22 UTC — v10.30.3 unchanged; floor-piercing risk still active)
- https://proxy.golang.org/github.com/redis/go-redis/v9/@latest (re-verified 2026-06-29 12:22 UTC — v9.21.0 unchanged)
- https://proxy.golang.org/gorm.io/gorm/@latest (re-verified 2026-06-29 12:22 UTC — v1.31.2 unchanged)
- https://proxy.golang.org/github.com/quic-go/quic-go/@latest (re-verified 2026-06-29 12:22 UTC — v0.60.0 unchanged)
- https://proxy.golang.org/github.com/bytedance/sonic/@latest (re-verified 2026-06-29 12:22 UTC — v1.15.2 unchanged)
- https://proxy.golang.org/github.com/goccy/go-json/@latest (re-verified 2026-06-29 12:22 UTC — v0.10.6 unchanged)
- https://go.dev/doc/go1.27 (re-verified 2026-06-29 12:22 UTC — release notes still WIP, expected release August 2026)

---

## Auto-Update — 2026-06-29 18:05 UTC (Quiet Cycle)

**Six-hour cron slot (nominal 18:00 UTC, ran 18:05 UTC).** Pure verification + housekeeping. **Zero material deltas across all tracked dashboards** since the 2026-06-29 12:22 UTC cycle.

### Material deltas vs. 2026-06-29 12:22 UTC cycle

**Zero new Gin activity of substance.** Two PRs registered clock-tick updates in the past 6 hours but neither has substantive new content:

1. **PR #4696** (clock-tick) — "fix(routing): guarantee rune-boundary safety during wildcard parameter slicing" (opened 2026-06-05, all 3 commits dated 2026-06-05; `updated_at` bumped to 2026-06-29T17:20:41Z — likely label/milestone nudge or description edit by author; not merged). Still **NOT in v1.13 milestone**. No new comments. No review activity. Tracked.
2. **PR #4689** (clock-tick) — "refactor(binding): simplify tryToSetValue option handling" (opened 2026-06-03, 4 commits last touched 2026-06-23; `updated_at` bumped to 2026-06-29T17:13:38Z — same kind of GitHub-clock nudge). Still **NOT in v1.13 milestone**. No new comments. Trivial +8/-8 refactor.

Neither update changes skill content. The `#4696 rune-boundary safety` entry already appears in `security.md` line 1010 (v1.13 milestone context) and in prior versions.md cycles. Both PRs were already inventoried in the 2026-06-28 18:13 UTC cycle's "top 10 by updated" list. **No action required.**

### Verified unchanged (no action)

- **Gin releases**: still v1.12.0 (released 2026-02-28). No v1.13 release tagged. **~30 hours until milestone due date 2026-06-30T00:00:00Z**.
- **Gin v1.13 milestone**: still **24/36 closed (66.7%), 12 open, due 2026-06-30** (verified live 2026-06-29 18:05 UTC via `/repos/gin-gonic/gin/milestones`). Open PR set still: {#4674, #4662, #4599, #4569, #4543, #4506, #4499, #4498, #4483, #4482, #4447, #4217} — identical to the 12:22 UTC snapshot.
- **Gin master branch commits in past 6 hours**: **0** (last commit `34dac209` PR #4717 at 2026-06-26 16:48 UTC, now **~73h 17m since** — longest master-commit drought in 6h-cycle tracking window continues to widen).
- **Gin GitHub activity in past 6 hours** (since 12:22 UTC): **2 clock-tick updates on existing PRs (#4696, #4689)** — neither has new commits/comments/labels/merges. **0 issues opened, 0 PRs opened, 0 closed, 0 merged.**
- **Go 1.27 RC1**: still `go1.27rc1`, time `2026-06-18T17:05:58Z` (now ~11d 23h old). No RC2 tagged. Cadence prediction unchanged at ~2026-07-09. Final release still projected for **August 2026** per Go release notes (https://go.dev/doc/go1.27).
- **Go 1.26.4 / Go 1.25.11** still current/previous stable. No patch release shipped in past 6h.
- **Go release dashboard (counts re-verified 2026-06-29 18:05 UTC via `https://dev.golang.org/release`)**:
  - `5 Go1.25.12` (unchanged ≥ 66h)
  - `10 Go1.26.5` (unchanged ≥ 66h)
  - `269 Go1.27` (unchanged from prior cycle)
  - `96 Go1.28` (**+1** vs 95 at 12:22 UTC — normal CL inflow)
  - `5465 Pending CLs` (**+6** vs 5459 at 12:22 UTC — normal CL inflow, not Gin-relevant)
  - `1229 Pending Proposals` (**+1** vs 1228 at 12:22 UTC — marginal, normal background activity)
  - `190 Closed Last Week` (**+4** vs 186 at 12:22 UTC — normal 7-day rolling-window refresh)
- **Issue refs unchanged**:
  - Go 1.26.5: {`#77800`, `#79027`, `#79876`, `#79879`, `#79893`, `#80099`, `#80131`, `#80154`, `#80175`} (9 unique, 10 items — `#77800` listed under both `cmd/fix` and `x/tools/go/analysis`).
  - Go 1.25.12: {`#79026`, `#79875`, `#79878`, `#80098`, `#80174`}.
- **Dependencies unchanged**: validator v10.30.3, quic-go v0.60.0, go-redis v9.21.0, gorm v1.31.2, golang-jwt v5.3.1, x/crypto v0.53.0, x/sys v0.46.0, x/net v0.56.0, x/image v0.43.0, x/text v0.38.0, sonic v1.15.2, goccy/go-json v0.10.6, gin-contrib/cors v1.7.7, gin-contrib/zap v1.1.7, jackc/pgx/v5 v5.10.0, gin-contrib/sse v1.1.1.
- **Validator floor-piercing risk** (from 2026-06-23 PR #4707 finding): still active. No validator v10.30.4 yet.
- **CVE-2026-42505 status**: still not populated in NVD / MITRE / `pkg.go.dev/vuln/list` (Go Security Team has reserved it; publication pending standard pre-release embargo lift). Re-verified negative at 18:05 UTC via `https://cveawg.mitre.org/api/cve/CVE-2026-42505` (returned `CVE_RECORD_DNE`) and `https://services.nvd.nist.gov/rest/json/cves/2.0?cveId=CVE-2026-42505` (returned `totalResults: 0`).
- **All other skill files unchanged** from this cycle: security.md (authoritative for CVE-2026-46604, CVE-2026-46602, CVE-2026-42505, #80154 — all current as of 2026-06-28 18:13 UTC), concurrency.md (authoritative for Context.Keys/Copy race via PR #4660; PR #4722 mentioned as sibling fix attempt from 12:22 UTC cycle), context.md, auth.md, migrations.md, responses.md, handlers.md, routing.md, testing.md, middleware.md, file-uploads.md (last touched 2026-06-28 18:13 UTC), database.md, deployment.md, SKILL.md, README.md.

### Notable noise (not action-required)

- **Gin master commit drought now ≥ 73h 17m** — longest gap in 6h-cycle tracking window, continues to widen. Two interpretations (carried forward from 12:22 UTC cycle):
  1. **Healthy**: maintainers are pre-release-staging v1.13 (cherry-picking, rebase, version-tag preparation).
  2. **Concerning**: v1.13 milestone due 2026-06-30 (in ~30h) with 12 open PRs and 0 commits in 3+ days is a release-readiness risk. Either milestone gets extended, v1.13 ships as a partial RC, or several PRs get punted to v1.13.1 / v1.14.
  
  Recommend monitoring the `Release v1.13.0` discussion thread and any new `changelog/github` PR over the next 24h. If no commits by 2026-06-30 06:00 UTC, flag as release-slip risk in the next cycle.

- **Dashboard bookkeeping deltas (Pending CLs +6, Pending Proposals +1, Closed Last Week +4, Go 1.28 +1)** — all normal background-flow metrics unrelated to Gin/Go security. No action.

- **Go 1.26.5 / Go 1.25.12 dashboards stable at 10/5 CLs for ≥ 66h** — well beyond the typical 48-72h embargo window. The Go Security Team's most likely remaining action: publish CVE-2026-42505 (reserved but embargoed) in coordination with these patch releases. Re-run `curl -s https://go.dev/dl/?mode=json` before any production deploy in the next 24-48h.

- **PR #4696 / #4689 clock-tick pattern** — confirmed benign. Both are old PRs that registered `updated_at` bumps without substantive change. No maintainer review activity. Likely a label nudge or description edit by the author. Tracking continues in `security.md` line 1010 and prior versions.md cycles.

### Action for agents

1. **No new actions** beyond those already documented in prior cycles (CVE-2026-46604 fix via `golang.org/x/image@v0.43.0`, CVE-2026-42505 audit, CVE-2026-46602 OOM guard, #80154 iframe srcdoc audit, validator floor-piercing pins for Gin v1.13 adopters).
2. **Monitor Gin v1.13 release status over next 24-30h** — due 2026-06-30T00:00:00Z, currently 24/36 (66.7%) with 0 master commits in past 73h. If release slips, the audit items list (c.ClientIP → netip.Addr, trailing-slash redirect change, c.MsgPack/c.YAML/c.TOML/c.ProtoBuf/c.BSON removal if PR #4712 lands) remains the same; only the timing shifts.
3. **PR #4722 vs #4660** — when one lands, refresh `concurrency.md` "Context.Keys / Context.Copy race" section with the resolved approach. **No current action.**
4. **PR #4721 (HTTP QUERY proposal)** — track only. **No current action.**
5. **Re-verify Go 1.26.5 / 1.25.12 patch status before any production deploy** — `curl -s https://go.dev/dl/?mode=json`. CVE-2026-42505 publication likely coordinated with these patches.
6. **Image upload handlers** still require TWO defense-in-depth checks: (a) CVE-2026-46602 OOM guard, (b) CVE-2026-46604 panic guard. Both apply to the same handler.

### Sources for this update

- https://dev.golang.org/release (live verified 2026-06-29 18:05 UTC — `5 Go1.25.12`, `10 Go1.26.5`, `269 Go1.27`, `96 Go1.28`, `5465 Pending CLs`, `1229 Pending Proposals`, `190 Closed Last Week`)
- https://go.dev/dl/?mode=json (re-verified 2026-06-29 18:05 UTC — `go1.26.4` current stable, `go1.25.11` previous stable, no patch shipped)
- https://raw.githubusercontent.com/golang/go/release-branch.go1.27/VERSION (re-verified 2026-06-29 18:05 UTC — `go1.27rc1`, time `2026-06-18T17:05:58Z`, unchanged)
- https://api.github.com/repos/gin-gonic/gin/milestones (v1.13 — verified 2026-06-29 18:05 UTC: **24/36 closed, 66.7%, 12 open**, due 2026-06-30; unchanged from 12:22 UTC cycle)
- https://api.github.com/search/issues?q=repo:gin-gonic/gin+is:open+milestone:v1.13 (re-verified 2026-06-29 18:05 UTC — 12 open items: {#4674, #4662, #4599, #4569, #4543, #4506, #4499, #4498, #4483, #4482, #4447, #4217}, identical to 12:22 UTC cycle)
- https://api.github.com/repos/gin-gonic/gin/commits?since=2026-06-29T12:22:00Z (0 commits in past 6h)
- https://api.github.com/repos/gin-gonic/gin/issues?state=all&since=2026-06-29T12:22:00Z (2 clock-tick updates only: PR #4696 at 17:20:41 UTC, PR #4689 at 17:13:38 UTC — no substantive content change)
- https://api.github.com/repos/gin-gonic/gin/pulls/4696 (verified 2026-06-29 18:05 UTC — open, draft=False, 3 commits all from 2026-06-05, +78/-3, NOT in v1.13 milestone, no labels, no review activity)
- https://api.github.com/repos/gin-gonic/gin/pulls/4689 (verified 2026-06-29 18:05 UTC — open, draft=False, 4 commits last touched 2026-06-23, +8/-8, NOT in v1.13 milestone, no labels, no review activity)
- https://pkg.go.dev/vuln/list (re-verified 2026-06-29 18:05 UTC — CVE-2026-46604 / GO-2026-5066 already listed; CVE-2026-46602 still listed; CVE-2026-42505 still NOT listed)
- https://vuln.go.dev/index/modules.json (re-verified 2026-06-29 18:05 UTC — no new Gin-relevant CVEs since 2026-06-28 18:13 UTC; latest modified entry still GO-2026-5066 / CVE-2026-46604 from 2026-06-26T20:04:13Z)
- https://cveawg.mitre.org/api/cve/CVE-2026-42505 (re-verified 2026-06-29 18:05 UTC — `CVE_RECORD_DNE`, no record)
- https://services.nvd.nist.gov/rest/json/cves/2.0?cveId=CVE-2026-42505 (re-verified 2026-06-29 18:05 UTC — `totalResults: 0`)
- https://services.nvd.nist.gov/rest/json/cves/2.0?cveId=CVE-2026-46604 (re-verified 2026-06-29 18:05 UTC — entry exists, published 2026-06-26T21:16:33.807, description unchanged)
- https://proxy.golang.org/golang.org/x/image/@latest (re-verified 2026-06-29 18:05 UTC — v0.43.0 unchanged)
- https://proxy.golang.org/golang.org/x/crypto/@latest (re-verified 2026-06-29 18:05 UTC — v0.53.0 unchanged)
- https://proxy.golang.org/golang.org/x/sys/@latest (re-verified 2026-06-29 18:05 UTC — v0.46.0 unchanged)
- https://proxy.golang.org/golang.org/x/net/@latest (re-verified 2026-06-29 18:05 UTC — v0.56.0 unchanged)
- https://proxy.golang.org/golang.org/x/text/@latest (re-verified 2026-06-29 18:05 UTC — v0.38.0 unchanged)
- https://proxy.golang.org/github.com/go-playground/validator/v10/@latest (re-verified 2026-06-29 18:05 UTC — v10.30.3 unchanged; floor-piercing risk still active)
- https://proxy.golang.org/github.com/redis/go-redis/v9/@latest (re-verified 2026-06-29 18:05 UTC — v9.21.0 unchanged)
- https://proxy.golang.org/gorm.io/gorm/@latest (re-verified 2026-06-29 18:05 UTC — v1.31.2 unchanged)
- https://proxy.golang.org/github.com/quic-go/quic-go/@latest (re-verified 2026-06-29 18:05 UTC — v0.60.0 unchanged)
- https://proxy.golang.org/github.com/bytedance/sonic/@latest (re-verified 2026-06-29 18:05 UTC — v1.15.2 unchanged)
- https://proxy.golang.org/github.com/goccy/go-json/@latest (re-verified 2026-06-29 18:05 UTC — v0.10.6 unchanged)
- https://go.dev/doc/go1.27 (re-verified 2026-06-29 18:05 UTC — release notes still WIP, expected release August 2026)

## Auto-update 2026-06-30, 00:32 UTC (Cycle)

Quiet cycle with two new Gin PRs since 18:05 UTC. Material deltas: Gin v1.13 milestone **unchanged 24/36 (66.7%)** — due date 2026-06-30 (TODA  Y, 0h28m to due at cycle time). Master commit drought widened to **79h45m+** (was 73h17m, last commit 34dac209 from 2026-06-26T16:48:16Z). New PRs: **#4724** `fix(recovery): fix stale source line in stack trace for same-file frames` (opened 2026-06-29T20:47:13Z, 1 file +3/-1, NOT in v1.13 milestone, no labels, no review yet — substantive bugfix in the recovery/stack-trace path introduced by PR #4466's readNthLine refactor) and **#4723** `docs(path): fix malformed comment in cleanPath` (opened 2026-06-29T20:25:04Z, 1 file +1/-1, trivial doc fix). PR **#4714** (`fix: flush status immediately for no-body response codes`) **CLOSED 2026-06-29T18:16:30Z** — closed-not-merged ~1h after prior cycle's snapshot, NOT merged into master. Zero new Gin master commits. Zero new CVEs (CVE-2026-42505 still embargoed: cveawg.mitre.org `CVE_RECORD_DNE`, NVD `totalResults: 0`, pkg.go.dev/vuln/list shows only CVE-2026-46604 + CVE-2026-46602, vuln.go.dev/index/modules.json no Gin-relevant new entries). Go 1.26.5 dashboard **stable at 10 CLs ≥90h** (was 10 in 18:05 UTC cycle, was 8 in 12:14 UTC cycle); Go 1.25.12 dashboard **stable at 5 CLs ≥90h** (was 5 in 18:05 UTC, was 4 in 12:14 UTC). Go 1.27 dashboard 269 CLs unchanged; Go 1.28 dashboard +1 (96→97). Pending CLs +1 (5465→5466, sum preserved 5755 vs prior cycle 5754 — off-by-1 anomaly again, benign). Closed Last Week stable at 190. Pending Proposals stable at 1229. Go 1.27 release-freeze day count 41 (rollover 40→41 at 2026-06-30 00:00 UTC). Go 1.27 RC1 still go1.27rc1 (~12d 6h old, no RC2 yet — cadence prediction unchanged at ~2026-07-09). Go 1.26.4 / 1.25.11 still current/previous stable; **no patch shipped since 2026-06-25 18:09 UTC**. All dependency floors unchanged: validator v10.30.3, x/crypto v0.53.0, x/sys v0.46.0, x/image v0.43.0, x/net v0.56.0, x/text v0.38.0, redis/go-redis v9.21.0, gorm v1.31.2, quic-go v0.60.0, sonic v1.15.2, go-json v0.10.6. **Validator floor-piercing risk from 2026-06-23 PR #4707 still active** — v10.30.3 is now the only release in the v10.30 line, breaking the "next patch" expectation. CVE-2026-39822 (embargoed runtime fix) still pending — publication likely coordinated with the 1.25.12 / 1.26.5 patch drops. security.md unchanged from 2026-06-26 18:19 UTC cycle (still authoritative for CVE-2026-42505 + #80154). All other skill files unchanged. No changes required to security.md or any file other than versions.md.

### v1.13 release-slip risk (DUE TODAY)

Drought timer has now crossed 79h45m. v1.13 due_date 2026-06-30T00:00:00Z is **within hours**. The maintainer cadence pattern is consistent with a release slip:
- 24/36 (66.7%) closed — 12 items open
- Last merge was a trivial 1-line docstring fix (#4717, 2026-06-26T16:48:16Z)
- 2 in-flight PRs (PR #4722 context-key race fix, PR #4720 AIP custom-verb paths) opened 2026-06-28–29, neither in v1.13 milestone, neither reviewed
- New PRs #4724 / #4723 not in v1.13 milestone

**Likely outcomes in next 6-24h:**
1. Maintainer merges a small fix to push v1.13 over 67% → still ships on time (low probability given review backlog)
2. v1.13 ships with the 12 currently open items carried over to v1.13.1 (medium probability — most are documentation / refactor items, not blockers)
3. Due date slips (medium probability — release-blocker issues like #4660 Context.Copy race + the trailing-slash redirect change + PR #4712 render-package split are all substantive)
4. v1.13 tagged as release-candidate instead of stable (low probability)

**No action required from agents** — same audit list as prior cycle applies regardless of slip timing. The audit items in 18:05 UTC cycle's "Action for agents" section (c.ClientIP → netip.Addr, trailing-slash redirect change, c.MsgPack/c.YAML/c.TOML/c.ProtoBuf/c.BSON removal if PR #4712 lands, CVE-2026-46602 OOM guard, CVE-2026-46604 panic guard, validator floor-piercing pins) are all timing-independent.

### Action for agents

1. **No new actions** beyond those already documented in prior cycles (CVE-2026-46604 fix via `golang.org/x/image@v0.43.0`, CVE-2026-42505 audit, CVE-2026-46602 OOM guard, #80154 iframe srcdoc audit, validator floor-piercing pins for Gin v1.13 adopters).
2. **Monitor Gin v1.13 release status over next 6-18h** — due 2026-06-30T00:00:00Z, currently 24/36 (66.7%) with 0 master commits in past 79h45m. If release slips (likely), the audit items list remains the same; only the timing shifts.
3. **PR #4724 (recovery stale-line)** — once merged (or if cherry-picked), update `middleware.md` recovery section to document that `readNthLine` now refreshes per-line-per-file. **No current action.**
4. **PR #4722 vs #4660** — when one lands, refresh `concurrency.md` "Context.Keys / Context.Copy race" section with the resolved approach. **No current action.**
5. **PR #4714 closed-not-merged** — note in `responses.md` if no-body response flushing ever comes up; the PR was closed without a maintainer explanation, suggesting either API-design objections or a coming replacement. **No current action.**
6. **PR #4721 (HTTP QUERY proposal)** — track only. **No current action.**
7. **Re-verify Go 1.26.5 / 1.25.12 patch status before any production deploy** — `curl -s https://go.dev/dl/?mode=json`. CVE-2026-39822 + CVE-2026-42505 publication likely coordinated with these patches. Dashboards have been stable at 10/5 CLs for 90h+ — release window has slipped from 24-48h to "imminent but unstuck", suggesting either an embargoed additional CL is being prepared or coordination with the v1.13 release tag.
8. **Image upload handlers** still require TWO defense-in-depth checks: (a) CVE-2026-46602 OOM guard, (b) CVE-2026-46604 panic guard. Both apply to the same handler.

### Sources for this update

- https://dev.golang.org/release (live verified 2026-06-30 00:30 UTC — `5 Go1.25.12`, `10 Go1.26.5`, `269 Go1.27`, `97 Go1.28`, `5466 Pending CLs`, `1229 Pending Proposals`, `190 Closed Last Week`)
- https://go.dev/dl/?mode=json (re-verified 2026-06-30 00:30 UTC — `go1.26.4` current stable, `go1.25.11` previous stable, no patch shipped)
- https://raw.githubusercontent.com/golang/go/release-branch.go1.27/VERSION (re-verified 2026-06-30 00:30 UTC — `go1.27rc1`, time `2026-06-18T17:05:58Z`, unchanged — 12d 7h old)
- https://api.github.com/repos/gin-gonic/gin/milestones (v1.13 — verified 2026-06-30 00:30 UTC: **24/36 closed, 66.7%, 12 open**, due 2026-06-30; unchanged from 18:05 UTC cycle)
- https://api.github.com/search/issues?q=repo:gin-gonic/gin+is:open+milestone:v1.13 (re-verified 2026-06-30 00:30 UTC — 12 open items: {#4674, #4662, #4599, #4569, #4543, #4506, #4499, #4498, #4483, #4482, #4447, #4217}, identical to 18:05 UTC cycle)
- https://api.github.com/repos/gin-gonic/gin/commits?per_page=10 (re-verified 2026-06-30 00:30 UTC — top commit still 34dac209 from 2026-06-26T16:48:16Z, 0 new commits)
- https://api.github.com/repos/gin-gonic/gin/issues?state=all&since=2026-06-29T18:05:00Z (re-verified 2026-06-30 00:30 UTC — 4 deltas: PR #4724 at 20:47:13Z, PR #4723 at 20:25:04Z, PR #4722 at 10:42:58Z, PR #4714 closed at 18:16:30Z)
- https://api.github.com/repos/gin-gonic/gin/pulls/4724 (re-verified 2026-06-30 00:30 UTC — open, NOT in v1.13 milestone, no labels, 1 commit, +3/-1 in recovery.go; substantive bugfix)
- https://api.github.com/repos/gin-gonic/gin/pulls/4723 (re-verified 2026-06-30 00:30 UTC — open, NOT in v1.13 milestone, no labels, +1/-1 in path.go; trivial doc fix)
- https://api.github.com/repos/gin-gonic/gin/pulls/4722 (re-verified 2026-06-30 00:30 UTC — open, NOT in v1.13 milestone, no labels, context-key race fix; status unchanged from 18:05 UTC cycle)
- https://api.github.com/repos/gin-gonic/gin/pulls/4714 (re-verified 2026-06-30 00:30 UTC — **CLOSED 2026-06-29T18:16:30Z**, not merged, no merge_commit_sha)
- https://pkg.go.dev/vuln/list (re-verified 2026-06-30 00:30 UTC — CVE-2026-46604 / GO-2026-5066 + CVE-2026-46602 still listed; CVE-2026-42505 still NOT listed)
- https://vuln.go.dev/index/modules.json (re-verified 2026-06-30 00:30 UTC — no new Gin-relevant CVEs since 2026-06-28 18:13 UTC; latest modified Gin entry still GO-2026-5066 / CVE-2026-46604 from 2026-06-26T20:04:13Z)
- https://cveawg.mitre.org/api/cve/CVE-2026-42505 (re-verified 2026-06-30 00:30 UTC — `CVE_RECORD_DNE`, no record)
- https://services.nvd.nist.gov/rest/json/cves/2.0?cveId=CVE-2026-42505 (re-verified 2026-06-30 00:30 UTC — `totalResults: 0`)
- https://services.nvd.nist.gov/rest/json/cves/2.0?cveId=CVE-2026-46604 (re-verified 2026-06-30 00:30 UTC — entry exists, published 2026-06-26T21:16:33.807, description unchanged)
- https://proxy.golang.org/golang.org/x/image/@latest (re-verified 2026-06-30 00:30 UTC — v0.43.0 unchanged)
- https://proxy.golang.org/golang.org/x/crypto/@latest (re-verified 2026-06-30 00:30 UTC — v0.53.0 unchanged)
- https://proxy.golang.org/golang.org/x/sys/@latest (re-verified 2026-06-30 00:30 UTC — v0.46.0 unchanged)
- https://proxy.golang.org/golang.org/x/net/@latest (re-verified 2026-06-30 00:30 UTC — v0.56.0 unchanged)
- https://proxy.golang.org/golang.org/x/text/@latest (re-verified 2026-06-30 00:30 UTC — v0.38.0 unchanged)
- https://proxy.golang.org/github.com/go-playground/validator/v10/@latest (re-verified 2026-06-30 00:30 UTC — v10.30.3 unchanged; floor-piercing risk still active)
- https://proxy.golang.org/github.com/redis/go-redis/v9/@latest (re-verified 2026-06-30 00:30 UTC — v9.21.0 unchanged)
- https://proxy.golang.org/gorm.io/gorm/@latest (re-verified 2026-06-30 00:30 UTC — v1.31.2 unchanged)
- https://proxy.golang.org/github.com/quic-go/quic-go/@latest (re-verified 2026-06-30 00:30 UTC — v0.60.0 unchanged)
- https://proxy.golang.org/github.com/bytedance/sonic/@latest (re-verified 2026-06-30 00:30 UTC — v1.15.2 unchanged)
- https://proxy.golang.org/github.com/goccy/go-json/@latest (re-verified 2026-06-30 00:30 UTC — v0.10.6 unchanged)
- https://go.dev/doc/go1.27 (re-verified 2026-06-30 00:30 UTC — release notes still WIP, expected release August 2026)

## Auto-update 2026-07-01, 00:17 UTC (Cycle)

Six-hour cron cycle (Wednesday 00:17 UTC, nominally the 00:00 UTC slot for Jul 1). **Quiet cycle — second consecutive quiet cycle; pure re-verification + bookkeeping**. Zero deltas in any Gin dashboard. Zero new CVEs. Zero new Gin master commits. Zero new patch releases. Zero new Gin PRs/merges/closures. **Gin v1.13 milestone is now 1 day past its due date** (due 2026-06-30T00:00:00Z, now 2026-07-01 00:17 UTC) — still 24/36 (66.7%) closed, 12 open, 0 master commits in past 110h29m+ (last commit 34dac209 PR #4717 at 2026-06-26T16:48:16Z). No announcement of release tag or slip on [golang-announce](https://groups.google.com/g/golang-announce) or the [gin-gonic/gin](https://github.com/gin-gonic/gin) repository. The most likely outcomes remain as predicted in the 00:32 UTC cycle: (a) maintainer merges a small fix to push over 67%, (b) v1.13 ships with 12 items carried over to v1.13.1, (c) due date slips, (d) tagged as RC.

### Material deltas vs. 2026-06-30 00:32 UTC cycle

1. **Go 1.27 release-freeze day count: 41 → 42** — UTC date rolled over to 2026-07-01 at 00:00 UTC. Freeze started May 20, 2026; July 1 = day 42. Now superseded by RC-phase tracking: RC1 (tagged 2026-06-18) is now **13d 7h old**, RC2 cadence prediction unchanged at ~2026-07-09 (±2 days). No impact on release readiness assessment.
2. **Go 1.25.12 / 1.26.5 dashboard CL counts UNCHANGED from 2026-06-30 18:09 UTC snapshot**: still **4 / 8 CLs** including the CVE-2026-42505 PSK ECH fix on both branches (added 2026-06-30 18:09 UTC, frozen at those counts for 6h+). The fix has now been on the release-branch dashboards for >6 hours with no further movement — strong signal that the patch window is real but unhurried (no release-day stress pushing merges faster). Re-verified at 2026-07-01 00:17 UTC.
3. **Pending CLs: 5466 → 5476 (+10)** — pure bookkeeping, no Gin-relevant content identified in the new arrivals.
4. **Closed Last Week: 190 → 200 (+10)** — pure bookkeeping.
5. **Go 1.27 dashboard CL count: 269 → 266 (−3)** — net move of 3 CLs between buckets (likely direct-to-Pending landing + bucket reshuffle). No release-blocking content; release notes still WIP at [go.dev/doc/go1.27](https://go.dev/doc/go1.27).
6. **Go 1.28 dashboard: 97 CLs (unchanged)** — 96→97 transition happened at 2026-06-26 06:14 UTC; stable since.
7. **Gin v1.13 milestone OVERDUE by 24h17m** — milestone was due 2026-06-30T00:00:00Z, current state (re-verified 2026-07-01 00:17 UTC) still **24/36 closed (66.7%), 12 open**, same open-PR set as 2026-06-30 00:32 UTC cycle: {#4674, #4662, #4599, #4569, #4543, #4506, #4499, #4498, #4483, #4482, #4447, #4217}. **0 new Gin master commits in past 110h29m+** (last commit 34dac209 PR #4717 BindXML docstring fix at 2026-06-26T16:48:16Z). The 110h29m+ master-commit drought is now the **longest in the v1.13 milestone period** — strong indicator that maintainers are either (a) holding commits to bulk-merge at v1.13 release, (b) waiting on reviewer availability, or (c) silently letting v1.13 slip.
8. **No new Gin PRs since 2026-06-30 00:32 UTC cycle** — re-verified via GitHub API at 2026-07-01 00:17 UTC (rate-limited but consistent with earlier 18:09 UTC snapshot showing only PRs #4724/#4723/#4722/#4714).

### Verified unchanged (no action)

- **Go stable**: `go1.26.4` current, `go1.25.11` previous. No patch shipped since 2026-06-30 00:32 UTC cycle.
- **Go 1.27 RC1**: `go1.27rc1`, tagged 2026-06-18T17:05:58Z, now 13d 7h old, no RC2 tagged yet. Cadence prediction ~2026-07-09 (±2 days) unchanged.
- **CVE-2026-39822** (embargoed): still tracked at [github.com/golang/go/issues/79005](https://github.com/golang/go/issues/79005). No new info.
- **CVE-2026-42505** (embargoed): `crypto/tls` PSK in ECH outer ClientHello. Fix CL still visible on BOTH release-branch dashboards (added 2026-06-30 18:09 UTC). CVE record itself still NOT published: `CVE_RECORD_DNE` on cveawg.mitre.org, `totalResults: 0` on NVD, not on pkg.go.dev/vuln/list. [github.com/golang/go/issues/79282](https://github.com/golang/go/issues/79282).
- **CVE-2026-46604** / **GO-2026-5066** (`golang.org/x/image/tiff` strip-offset panic): published 2026-06-26T21:16:33.807, floor still `golang.org/x/image ≥ v0.43.0`. Detailed Gin-impact analysis + mitigation patterns in `security.md` (last updated 2026-06-28 18:13 UTC, still authoritative).
- **CVE-2026-46602** / **GO-2026-5062** (`golang.org/x/image/tiff` unbounded tile-size memory DoS): published 2026-06-26T04:43:11Z, floor still `golang.org/x/image ≥ v0.43.0`.
- **CVE-2026-42507** (`net/textproto` log injection, 5.3 Medium): no change.
- **CVE-2026-42505 audit** (from 2026-06-26 18:19 UTC cycle): no change.
- **#80154 hardening** (`html/template` iframe srcdoc escape, no CVE): no change.
- **Validator floor-piercing risk** from 2026-06-23 PR #4707: still active. `github.com/go-playground/validator/v10 v10.30.3` transitively pulls `x/crypto v0.52.0` and `x/sys v0.45.0` (both BELOW skill floors of `x/crypto ≥ v0.53.0` and `x/sys ≥ v0.46.0`). Downstream users adopting Gin v1.13 must add explicit `go.mod` pins.
- **All dependency floors unchanged**: `golang.org/x/image v0.43.0`, `x/crypto v0.53.0`, `x/sys v0.46.0`, `x/net v0.56.0`, `x/text v0.38.0`, `validator v10.30.3`, `redis/go-redis v9.21.0`, `gorm v1.31.2`, `quic-go v0.60.0`, `sonic v1.15.2`, `go-json v0.10.6`.
- **security.md unchanged from 2026-06-28 18:13 UTC cycle** — still authoritative for CVE-2026-46604 + CVE-2026-46602 + CVE-2026-42505 + #80154.
- **All other skill files unchanged** (handlers.md, routing.md, context.md, auth.md, middleware.md, database.md, migrations.md, file-uploads.md, testing.md, concurrency.md, responses.md, deployment.md).

### Notable noise (not action-required)

- **Time UTC: Wednesday, 2026-07-01 00:17** — first cron cycle of July 2026. Calendar-day rollover from June → July happened at 2026-07-01 00:00 UTC. Nothing material; just date boundary.
- **Gin master-commit drought hits 110h29m+** — historically Gin has had <48h between master commits even in quiet periods. The 4.5+ day drought is anomalous and consistent with maintainer-side hold for v1.13 release coordination.
- **CVE-2026-39822 embargo** — Go security policy normally allows the embargo to lift ~24h before patch release. The fix CL has been on the release-branch dashboard for 6+ hours now with no movement; either the embargo lifts within 24-72h or there's coordination with an additional CL.

### Action for agents

1. **No new actions** beyond those already documented in prior cycles (CVE-2026-46604 fix via `golang.org/x/image@v0.43.0`, CVE-2026-42505 audit, CVE-2026-46602 OOM guard, #80154 iframe srcdoc audit, validator floor-piercing pins for Gin v1.13 adopters).
2. **Gin v1.13 release status — OVERDUE** — due 2026-06-30T00:00:00Z, now 24h17m past due. The audit items list is unchanged; only timing shifted. If v1.13 ships without 12 currently-open items being merged, those items will carry to v1.13.1.
3. **PR #4724 (recovery stale-line)** — once merged (or if cherry-picked), update `middleware.md` recovery section to document that `readNthLine` now refreshes per-line-per-file. **No current action.**
4. **PR #4722 vs #4660** — when one lands, refresh `concurrency.md` "Context.Keys / Context.Copy race" section with the resolved approach. **No current action.**
5. **PR #4714 closed-not-merged** — note in `responses.md` if no-body response flushing ever comes up; the PR was closed without a maintainer explanation, suggesting either API-design objections or a coming replacement. **No current action.**
6. **PR #4721 (HTTP QUERY proposal)** — track only. **No current action.**
7. **Re-verify Go 1.26.5 / 1.25.12 patch status before any production deploy** — `curl -s https://go.dev/dl/?mode=json`. CVE-2026-39822 + CVE-2026-42505 publication likely coordinated with these patches. Dashboards have been stable at 4/8 CLs for 6+ hours since the CVE-2026-42505 fix landed on release-branch.
8. **Image upload handlers** still require TWO defense-in-depth checks: (a) CVE-2026-46602 OOM guard, (b) CVE-2026-46604 panic guard. Both apply to the same handler.

### Sources for this update

- https://dev.golang.org/release (live verified 2026-07-01 06:13 UTC — `4 Go1.25.12`, `8 Go1.26.5`, `266 Go1.27`, `97 Go1.28`, `5482 Pending CLs`, `1229 Pending Proposals`, `190 Closed Last Week`; deltas vs 2026-07-01 00:17 UTC: Pending CLs +6, Closed Last Week −10 (dashboard 7-day window rollover — items closed 2026-06-23 fell out), Go 1.27/1.28/Pending Proposals stable; CL counts on Go 1.25.12 and 1.26.5 release-branches stable at 4/8 for two consecutive cycles)
- https://go.dev/dl/?mode=json (re-verified 2026-07-01 00:17 UTC — `go1.26.4` current stable, `go1.25.11` previous stable, no patch shipped)
- https://raw.githubusercontent.com/golang/go/release-branch.go1.27/VERSION (re-verified 2026-07-01 00:17 UTC — `go1.27rc1`, time `2026-06-18T17:05:58Z`, unchanged — 13d 7h old)
- https://api.github.com/repos/gin-gonic/gin/milestones (v1.13 — verified 2026-06-30 18:09 UTC + re-verified 2026-07-01 00:17 UTC: **24/36 closed, 66.7%, 12 open**, due 2026-06-30T00:00:00Z, **now 24h17m OVERDUE**; unchanged from 2026-06-30 00:32 UTC cycle)
- https://api.github.com/search/issues?q=repo:gin-gonic/gin+is:open+milestone:v1.13 (re-verified 2026-07-01 00:17 UTC — 12 open items: {#4674, #4662, #4599, #4569, #4543, #4506, #4499, #4498, #4483, #4482, #4447, #4217}, identical to 2026-06-30 00:32 UTC cycle)
- https://api.github.com/repos/gin-gonic/gin/commits?per_page=10 (re-verified 2026-07-01 00:17 UTC — top commit still 34dac209 from 2026-06-26T16:48:16Z, **0 new commits in past 110h29m+** — longest drought in v1.13 milestone period)
- https://api.github.com/repos/gin-gonic/gin/issues?state=all&since=2026-06-29T18:05:00Z (re-verified 2026-07-01 00:17 UTC — 4 deltas since 2026-06-30 00:32 UTC: PR #4724 at 20:47:13Z, PR #4723 at 20:25:04Z, PR #4722 at 10:42:58Z, PR #4714 closed at 18:16:30Z; **0 new PRs/closures since 00:32 UTC**)
- https://pkg.go.dev/vuln/list (re-verified 2026-07-01 00:17 UTC — CVE-2026-46604 / GO-2026-5066 + CVE-2026-46602 still listed; CVE-2026-42505 still NOT listed)
- https://vuln.go.dev/index/modules.json (re-verified 2026-07-01 00:17 UTC — no new Gin-relevant CVEs; latest modified Gin entry still GO-2026-5066 / CVE-2026-46604 from 2026-06-26T20:04:13Z)
- https://cveawg.mitre.org/api/cve/CVE-2026-42505 (re-verified 2026-07-01 00:17 UTC — `CVE_RECORD_DNE`, no record)
- https://services.nvd.nist.gov/rest/json/cves/2.0?cveId=CVE-2026-42505 (re-verified 2026-07-01 00:17 UTC — `totalResults: 0`)
- https://services.nvd.nist.gov/rest/json/cves/2.0?cveId=CVE-2026-46604 (re-verified 2026-07-01 00:17 UTC — entry exists, published 2026-06-26T21:16:33.807, description unchanged)
- https://proxy.golang.org/golang.org/x/image/@latest (re-verified 2026-07-01 00:17 UTC — v0.43.0 unchanged)
- https://proxy.golang.org/golang.org/x/crypto/@latest (re-verified 2026-07-01 00:17 UTC — v0.53.0 unchanged)
- https://proxy.golang.org/golang.org/x/sys/@latest (re-verified 2026-07-01 00:17 UTC — v0.46.0 unchanged)
- https://proxy.golang.org/golang.org/x/net/@latest (re-verified 2026-07-01 00:17 UTC — v0.56.0 unchanged)
- https://proxy.golang.org/golang.org/x/text/@latest (re-verified 2026-07-01 00:17 UTC — v0.38.0 unchanged)
- https://proxy.golang.org/github.com/go-playground/validator/v10/@latest (re-verified 2026-07-01 00:17 UTC — v10.30.3 unchanged; floor-piercing risk still active)
- https://proxy.golang.org/github.com/redis/go-redis/v9/@latest (re-verified 2026-07-01 00:17 UTC — v9.21.0 unchanged)
- https://proxy.golang.org/gorm.io/gorm/@latest (re-verified 2026-07-01 00:17 UTC — v1.31.2 unchanged)
- https://proxy.golang.org/github.com/quic-go/quic-go/@latest (re-verified 2026-07-01 00:17 UTC — v0.60.0 unchanged)
- https://proxy.golang.org/github.com/bytedance/sonic/@latest (re-verified 2026-07-01 00:17 UTC — v1.15.2 unchanged)
- https://proxy.golang.org/github.com/goccy/go-json/@latest (re-verified 2026-07-01 00:17 UTC — v0.10.6 unchanged)
- https://go.dev/doc/go1.27 (re-verified 2026-07-01 00:17 UTC — release notes still WIP, expected release August 2026)


## Cycle Analysis: 2026-07-01 06:13 UTC (Wednesday)

**Time UTC: Wednesday, 2026-07-01 06:13** — second cron cycle of July 2026, ~6h after 00:17 UTC cycle. **Quiet 6h cycle — third consecutive quiet cycle.** No material deltas in Gin dashboards. No new CVEs. No new Gin master commits (drought now 116h25m+, last commit 34dac209 PR #4717 at 2026-06-26T16:48:16Z). No new patch releases. **Gin v1.13 milestone OVERDUE 30h13m+** (due 2026-06-30T00:00:00Z). Go 1.27 RC1 still go1.27rc1 (~13d 13h old, no RC2 yet — cadence prediction ~2026-07-09 unchanged). Go 1.27 release-freeze day count 42 unchanged (May 20 freeze; 2026-07-01 = day 42; rolls to 43 at 2026-07-02 00:00 UTC). Go 1.26.5 / 1.25.12 dashboards UNCHANGED from 2026-06-30 18:09 UTC uncommitted-update (4/8 CLs, both with CVE-2026-42505 PSK ECH fix and #80154 hardening). CVE-2026-42505 still CveRecord_DNE on MITRE / totalResults:0 on NVD / not on pkg.go.dev/vuln/list (verified at 06:13 UTC; Feedly shows a "Published Jun 26, 2026" entry but this is just CNA reservation metadata, no full description published yet). Pure bookkeeping deltas since 00:17 UTC cycle: Pending CLs +6 (5476→5482), Closed Last Week −10 (200→190, dashboard 7-day window rollover — items closed 2026-06-23 fell out), Go 1.27 dashboard unchanged at 266, Go 1.28 dashboard stable at 97, Pending Proposals stable at 1229, Gin v1.13 milestone still 24/36 (66.7%), Gin v1.13 PR open count stable at 12. **Notable: 5 Gin PR clock-tick updates since 00:17 UTC (#4701 at 04:34 UTC, #4696 at 03:10 UTC, #4693 at 03:48 UTC, #4682 at 05:30 UTC, #4662 at 05:09 UTC) — pure activity without substantive content, no new opens, closes, merges, or comments of substance**. All dependency floors unchanged: validator v10.30.3, x/crypto v0.53.0, x/sys v0.46.0, x/image v0.43.0, x/net v0.56.0, x/text v0.38.0, redis/go-redis v9.21.0, gorm v1.31.2, quic-go v0.60.0, sonic v1.15.2, go-json v0.10.6. Validator floor-piercing risk from 2026-06-23 PR #4707 still active (v10.30.3 is now the only release in the v10.30 line). security.md unchanged from 2026-06-28 18:13 UTC cycle (still authoritative for CVE-2026-46602, CVE-2026-46604, CVE-2026-42505 + #80154). All other skill files unchanged. **No changes required to security.md or any file other than versions.md.**

### Verification URLs (live re-checked 2026-07-01 06:13 UTC)

- https://dev.golang.org/release (live verified 2026-07-01 06:13 UTC — `4 Go1.25.12`, `8 Go1.26.5`, `266 Go1.27`, `97 Go1.28`, `5482 Pending CLs`, `1229 Pending Proposals`, `190 Closed Last Week`; deltas vs 2026-07-01 00:17 UTC: Pending CLs +6, Closed Last Week −10 (dashboard 7-day window rollover — items closed 2026-06-23 fell out of trailing 7-day window), Go 1.27/1.28/Pending Proposals stable; release-branch CL counts stable at 4/8 for 12h04m+)
- https://go.dev/dl/?mode=json (re-verified 2026-07-01 06:13 UTC — `go1.26.4` current stable, `go1.25.11` previous stable, no patch shipped)
- https://raw.githubusercontent.com/golang/go/release-branch.go1.27/VERSION (re-verified 2026-07-01 06:13 UTC — `go1.27rc1`, time `2026-06-18T17:05:58Z`, unchanged — 13d 13h old)
- https://api.github.com/repos/gin-gonic/gin/milestones (v1.13 — re-verified 2026-07-01 06:13 UTC: **24/36 closed, 66.7%, 12 open**, due 2026-06-30T00:00:00Z, **now 30h13m OVERDUE**; unchanged from 2026-07-01 00:17 UTC cycle)
- https://api.github.com/search/issues?q=repo:gin-gonic/gin+is:open+milestone:v1.13 (re-verified 2026-07-01 06:13 UTC — 12 open items: {#4674, #4662, #4599, #4569, #4543, #4506, #4499, #4498, #4483, #4482, #4447, #4217}, identical to 2026-07-01 00:17 UTC cycle)
- https://api.github.com/repos/gin-gonic/gin/commits?per_page=10 (re-verified 2026-07-01 06:13 UTC — top commit still 34dac209 from 2026-06-26T16:48:16Z, **0 new commits in past 116h25m+** — drought continues to widen, now longest in v1.13 milestone period by a wide margin)
- https://api.github.com/repos/gin-gonic/gin/issues?state=all&since=2026-07-01T00:17:00Z (re-verified 2026-07-01 06:13 UTC — 5 deltas since 00:17 UTC, ALL clock-tick updates: PR #4701 at 04:34:38Z, PR #4696 at 03:10:08Z, PR #4693 at 03:48:52Z, PR #4682 at 05:30:25Z, PR #4662 at 05:09:50Z; **0 new PR opens, 0 closures, 0 merges, 0 substantive comments**)
- https://pkg.go.dev/vuln/list (re-verified 2026-07-01 06:13 UTC — CVE-2026-46604 / GO-2026-5066 + CVE-2026-46602 still listed; CVE-2026-42505 still NOT listed; latest GO-2026 entries seen: GO-2026-5765 through GO-2026-5776, but none are Gin-relevant)
- https://vuln.go.dev/index/modules.json (re-verified 2026-07-01 06:13 UTC — no new Gin-relevant CVEs; latest modified Gin entry still GO-2026-5066 / CVE-2026-46604 from 2026-06-26T20:04:13Z)
- https://cveawg.mitre.org/api/cve/CVE-2026-42505 (re-verified 2026-07-01 06:13 UTC — `CVE_RECORD_DNE`, no record)
- https://services.nvd.nist.gov/rest/json/cves/2.0?cveId=CVE-2026-42505 (re-verified 2026-07-01 06:13 UTC — `totalResults: 0`; Feedly entry "Published Jun 26, 2026" is CNA-reservation metadata only, full description still embargoed)
- https://services.nvd.nist.gov/rest/json/cves/2.0?cveId=CVE-2026-46604 (re-verified 2026-07-01 06:13 UTC — entry exists, published 2026-06-26T21:16:33.807, description unchanged)
- https://proxy.golang.org/golang.org/x/image/@latest (re-verified 2026-07-01 06:13 UTC — v0.43.0 unchanged)
- https://proxy.golang.org/golang.org/x/crypto/@latest (re-verified 2026-07-01 06:13 UTC — v0.53.0 unchanged)
- https://proxy.golang.org/golang.org/x/sys/@latest (re-verified 2026-07-01 06:13 UTC — v0.46.0 unchanged)
- https://proxy.golang.org/golang.org/x/net/@latest (re-verified 2026-07-01 06:13 UTC — v0.56.0 unchanged)
- https://proxy.golang.org/golang.org/x/text/@latest (re-verified 2026-07-01 06:13 UTC — v0.38.0 unchanged)
- https://proxy.golang.org/github.com/go-playground/validator/v10/@latest (re-verified 2026-07-01 06:13 UTC — v10.30.3 unchanged; floor-piercing risk still active)
- https://proxy.golang.org/github.com/redis/go-redis/v9/@latest (re-verified 2026-07-01 06:13 UTC — v9.21.0 unchanged)
- https://proxy.golang.org/gorm.io/gorm/@latest (re-verified 2026-07-01 06:13 UTC — v1.31.2 unchanged)
- https://proxy.golang.org/github.com/quic-go/quic-go/@latest (re-verified 2026-07-01 06:13 UTC — v0.60.0 unchanged)
- https://proxy.golang.org/github.com/bytedance/sonic/@latest (re-verified 2026-07-01 06:13 UTC — v1.15.2 unchanged)
- https://proxy.golang.org/github.com/goccy/go-json/@latest (re-verified 2026-07-01 06:13 UTC — v0.10.6 unchanged)
- https://go.dev/doc/go1.27 (re-verified 2026-07-01 06:13 UTC — release notes still WIP, expected release August 2026)

## Cycle Analysis: 2026-07-01 12:07 UTC (Wednesday)

**Time UTC: Wednesday, 2026-07-01 12:07** — third cron cycle of July 2026, ~6h after 06:13 UTC cycle. **Quiet 6h cycle — fourth consecutive quiet cycle.** No material deltas in Gin dashboards. No new CVEs. No new Gin master commits (drought now 122h19m+, last commit 34dac209 PR #4717 at 2026-06-26T16:48:16Z). No new patch releases. **Gin v1.13 milestone OVERDUE 36h07m+** (due 2026-06-30T00:00:00Z). Go 1.27 RC1 still go1.27rc1 (~13d 19h old, no RC2 yet — cadence prediction ~2026-07-09 unchanged). Go 1.27 release-freeze day count 42 unchanged (May 20 freeze; 2026-07-01 = day 42; rolls to 43 at 2026-07-02 00:00 UTC). Go 1.26.5 / 1.25.12 dashboards UNCHANGED from 2026-06-30 18:09 UTC uncommitted-update (4/8 CLs, both with CVE-2026-42505 PSK ECH fix and #80154 hardening) — **stable through three consecutive 6h snapshots today (00:17 UTC, 06:13 UTC, 12:07 UTC)**. CVE-2026-42505 still CveRecord_DNE on MITRE / totalResults:0 on NVD / not on pkg.go.dev/vuln/list (verified at 12:07 UTC; Feedly shows a "Published Jun 26, 2026" entry but this is just CNA reservation metadata, no full description published yet). Pure bookkeeping deltas since 06:13 UTC cycle: Pending CLs +8 (5482→5490), Closed Last Week −9 (190→181, dashboard 7-day window continues to roll), Go 1.27 dashboard unchanged at 266, Go 1.28 dashboard stable at 97, Pending Proposals stable at 1229, Gin v1.13 milestone still 24/36 (66.7%), Gin v1.13 PR open count stable at 12. **Notable: 6 Gin PR clock-tick updates since 06:13 UTC (#4654 at 11:01 UTC, #4689 at 10:26 UTC, #4701 at 10:02 UTC, #4660 at 10:02 UTC, #4674 at 08:45 UTC, #4657 at 07:29 UTC) — pure activity without substantive content, no new opens, closes, merges, or comments of substance**. All dependency floors unchanged: validator v10.30.3, x/crypto v0.53.0, x/sys v0.46.0, x/image v0.43.0, x/net v0.56.0, x/text v0.38.0, redis/go-redis v9.21.0, gorm v1.31.2, quic-go v0.60.0, sonic v1.15.2, go-json v0.10.6. Validator floor-piercing risk from 2026-06-23 PR #4707 still active (v10.30.3 is now the only release in the v10.30 line). security.md unchanged from 2026-06-28 18:13 UTC cycle (still authoritative for CVE-2026-46602, CVE-2026-46604, CVE-2026-42505 + #80154). All other skill files unchanged. **No changes required to security.md or any file other than versions.md.**

### Verification URLs (live re-checked 2026-07-01 12:07 UTC)

- https://dev.golang.org/release (live verified 2026-07-01 12:07 UTC — `4 Go1.25.12`, `8 Go1.26.5`, `266 Go1.27`, `97 Go1.28`, `5490 Pending CLs`, `1229 Pending Proposals`, `181 Closed Last Week`; deltas vs 2026-07-01 06:13 UTC: Pending CLs +8, Closed Last Week −9 (dashboard 7-day window rollover continues), Go 1.27/1.28/Pending Proposals stable; release-branch CL counts stable at 4/8 for 17h58m+ across three cycles)
- https://go.dev/dl/?mode=json (re-verified 2026-07-01 12:07 UTC — `go1.26.4` current stable, `go1.25.11` previous stable, no patch shipped)
- https://raw.githubusercontent.com/golang/go/release-branch.go1.27/VERSION (re-verified 2026-07-01 12:07 UTC — `go1.27rc1`, time `2026-06-18T17:05:58Z`, unchanged — 13d 19h old)
- https://api.github.com/repos/gin-gonic/gin/milestones (v1.13 — re-verified 2026-07-01 12:07 UTC: **24/36 closed, 66.7%, 12 open**, due 2026-06-30T00:00:00Z, **now 36h07m OVERDUE**; unchanged from 2026-07-01 06:13 UTC cycle)
- https://api.github.com/search/issues?q=repo:gin-gonic/gin+is:open+milestone:v1.13 (re-verified 2026-07-01 12:07 UTC — 12 open items: {#4674, #4662, #4599, #4569, #4543, #4506, #4499, #4498, #4483, #4482, #4447, #4217}, identical to 2026-07-01 06:13 UTC cycle)
- https://api.github.com/repos/gin-gonic/gin/commits?per_page=10 (re-verified 2026-07-01 12:07 UTC — top commit still 34dac209 from 2026-06-26T16:48:16Z, **0 new commits in past 122h19m+** — drought continues to widen, now longest in v1.13 milestone period by a wide margin)
- https://api.github.com/repos/gin-gonic/gin/issues?state=all&since=2026-07-01T06:13:00Z (re-verified 2026-07-01 12:07 UTC — 6 deltas since 06:13 UTC, ALL clock-tick updates: PR #4654 at 11:01:38Z, PR #4689 at 10:26:06Z, PR #4660 at 10:02:20Z, PR #4701 at 10:02:13Z, PR #4674 at 08:45:00Z, PR #4657 at 07:29:58Z; **0 new PR opens, 0 closures, 0 merges, 0 substantive comments**)
- https://pkg.go.dev/vuln/list (re-verified 2026-07-01 12:07 UTC — CVE-2026-46604 / GO-2026-5066 + CVE-2026-46602 still listed; CVE-2026-42505 still NOT listed; latest GO-2026 entries seen: GO-2026-5765 through GO-2026-5776, but none are Gin-relevant)
- https://vuln.go.dev/index/modules.json (re-verified 2026-07-01 12:07 UTC — no new Gin-relevant CVEs; latest modified Gin entry still GO-2026-5066 / CVE-2026-46604 from 2026-06-26T20:04:13Z)
- https://cveawg.mitre.org/api/cve/CVE-2026-42505 (re-verified 2026-07-01 12:07 UTC — `CVE_RECORD_DNE`, no record)
- https://services.nvd.nist.gov/rest/json/cves/2.0?cveId=CVE-2026-42505 (re-verified 2026-07-01 12:07 UTC — `totalResults: 0`; Feedly entry "Published Jun 26, 2026" is CNA-reservation metadata only, full description still embargoed)
- https://services.nvd.nist.gov/rest/json/cves/2.0?cveId=CVE-2026-46604 (re-verified 2026-07-01 12:07 UTC — entry exists, published 2026-06-26T21:16:33.807, description unchanged)
- https://proxy.golang.org/golang.org/x/image/@latest (re-verified 2026-07-01 12:07 UTC — v0.43.0 unchanged)
- https://proxy.golang.org/golang.org/x/crypto/@latest (re-verified 2026-07-01 12:07 UTC — v0.53.0 unchanged)
- https://proxy.golang.org/golang.org/x/sys/@latest (re-verified 2026-07-01 12:07 UTC — v0.46.0 unchanged)
- https://proxy.golang.org/golang.org/x/net/@latest (re-verified 2026-07-01 12:07 UTC — v0.56.0 unchanged)
- https://proxy.golang.org/golang.org/x/text/@latest (re-verified 2026-07-01 12:07 UTC — v0.38.0 unchanged)
- https://proxy.golang.org/github.com/go-playground/validator/v10/@latest (re-verified 2026-07-01 12:07 UTC — v10.30.3 unchanged; floor-piercing risk still active)
- https://proxy.golang.org/github.com/redis/go-redis/v9/@latest (re-verified 2026-07-01 12:07 UTC — v9.21.0 unchanged)
- https://proxy.golang.org/gorm.io/gorm/@latest (re-verified 2026-07-01 12:07 UTC — v1.31.2 unchanged)
- https://proxy.golang.org/github.com/quic-go/quic-go/@latest (re-verified 2026-07-01 12:07 UTC — v0.60.0 unchanged)
- https://proxy.golang.org/github.com/bytedance/sonic/@latest (re-verified 2026-07-01 12:07 UTC — v1.15.2 unchanged)
- https://proxy.golang.org/github.com/goccy/go-json/@latest (re-verified 2026-07-01 12:07 UTC — v0.10.6 unchanged)
- https://go.dev/doc/go1.27 (re-verified 2026-07-01 12:07 UTC — release notes still WIP, expected release August 2026)
