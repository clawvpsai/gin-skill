# Security — Headers, CORS, Rate Limiting, Hardening, Recent CVEs

## Security Headers

```go
func SecurityHeaders() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Header("X-Content-Type-Options", "nosniff")
        c.Header("X-Frame-Options", "DENY")
        c.Header("Referrer-Policy", "strict-origin-when-cross-origin")
        c.Header("Content-Security-Policy", "default-src 'self'")
        c.Header("Permissions-Policy", "geolocation=(), microphone=(), camera=()")
        c.Header("Strict-Transport-Security", "max-age=31536000; includeSubDomains; preload")
        c.Next()
    }
}

r.Use(SecurityHeaders())
```

**Note:** `X-XSS-Protection` is removed from modern browsers — do not include it. CSP is the proper XSS protection. Also added `preload` to HSTS for HSTS preload list submission.

## ⚠️ Critical Production Server Setup (CVE-2026-52878 Mitigation)

**`gin.Engine.Run(":8080")` is unsafe for production.** It uses Go's default `http.Server` with **no timeouts**, leaving the server vulnerable to **slow-header / slowloris DoS attacks** that exhaust file descriptors. This was confirmed in production via CVE-2026-52878 / GHSA-w4c6-7r69-w7j9 (June 2026) against klever-go's Gin-based REST API.

**Always use `http.Server` directly with explicit timeouts in production:**

```go
func NewProductionServer(addr string, r *gin.Engine) *http.Server {
    return &http.Server{
        Addr:    addr,
        Handler: r,

        // CVE-2026-52878 mitigation: limit slow-header / slowloris attacks
        ReadHeaderTimeout: 5 * time.Second,  // max time to read request headers
        ReadTimeout:       30 * time.Second, // total request read time (headers + body)
        WriteTimeout:      30 * time.Second, // response write deadline
        IdleTimeout:       120 * time.Second, // keep-alive idle timeout

        // Header size cap (defends against memory exhaustion via large headers)
        MaxHeaderBytes: 1 << 20, // 1 MB
    }
}

func main() {
    r := gin.New()
    r.Use(gin.Recovery(), SecurityHeaders(), RequestSizeLimit(10<<20))

    r.GET("/health", func(c *gin.Context) {
        c.JSON(200, gin.H{"status": "ok"})
    })

    srv := NewProductionServer(":8080", r)

    // Graceful shutdown on SIGTERM/SIGINT
    go func() {
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("listen: %v", err)
        }
    }()

    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit
    log.Println("shutdown requested")

    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    if err := srv.Shutdown(ctx); err != nil {
        log.Printf("graceful shutdown failed: %v", err)
    }
}
```

**Why this matters:**
- `gin.Engine.Run(addr)` calls `http.ListenAndServe(addr, engine)` — this uses `http.DefaultServer` with **no timeouts** (deprecated since Go 1.8).
- An attacker can open thousands of TCP connections, send partial HTTP headers very slowly, and exhaust the file descriptor pool — taking down the entire API.
- `ReadHeaderTimeout` is the critical one. Even setting just that to 5–10s mitigates the slow-header attack.

**Alternative — use `gin-contrib/timeout` middleware for per-handler timeouts:**
```go
import "github.com/gin-contrib/timeout"

r.Use(timeout.New(
    timeout.WithTimeout(10*time.Second),
    timeout.WithHandler(func(c *gin.Context) {
        c.JSON(http.StatusRequestTimeout, gin.H{"error": "timeout"})
    }),
    timeout.WithResponse(func(c *gin.Context) {
        c.JSON(http.StatusRequestTimeout, gin.H{"error": "request timed out"})
    }),
))
```

## Recent Gin/Go CVEs (May–June 2026)

### CVE-2026-52878 / GHSA-w4c6-7r69-w7j9 — Gin Engine slow-header DoS
- **Published:** 2026-06-05 | **Severity:** High
- **Affected:** Any Gin app using `gin.Engine.Run(addr)` or `http.ListenAndServe()` without explicit `http.Server` timeouts
- **Impact:** Remote, unauthenticated attacker opens many TCP connections sending partial HTTP headers extremely slowly, exhausting file descriptors and causing complete API unavailability.
- **Fix:** Use `http.Server` with `ReadHeaderTimeout`, `ReadTimeout`, `WriteTimeout`, `IdleTimeout` (see code above). `r.Run(":8080")` is for development only.
- **Detection:** Monitor `lsof -p $PID | wc -l` for FD count spikes; `netstat -an | grep <port> | grep ESTABLISHED | wc -l` for connection count.

### CVE-2026-42507 — Go `net/textproto` log injection
- **Published:** 2026-06-04 | **Severity:** Medium
- **Affected:** All Go HTTP servers including Gin — affects any handler that logs `textproto.Error()` output or header parse errors.
- **Impact:** Attacker-controlled input included in error messages without escaping. Enables misleading log entries, terminal-control injection (ANSI escapes), and log injection attacks when logs are ingested by monitoring tools.
- **Fix:** Patched in upcoming Go 1.25.12 / 1.26.5. Until patched, **sanitize HTTP header values before logging** — use `strconv.Quote()` or strip non-printable chars before `log.Printf()`.
- **Workaround example:**
  ```go
  // UNSAFE — raw header in logs
  log.Printf("bad header from %s: %v", c.Request.RemoteAddr, err)

  // SAFER — quote header values to neutralize control chars
  log.Printf("bad header from %s: %s",
      c.Request.RemoteAddr,
      strconv.Quote(c.GetHeader("X-Custom")))
  ```

### CVE-2026-42500 — Go `image` package BMP paletted-image panic
- **Published:** 2026-05-29 | **Severity:** Medium
- **Affected:** Image upload handlers using `golang.org/x/image/bmp` (or code paths that fall back to BMP decoding via `image.Decode`).
- **Impact:** Decoding a paletted BMP with an out-of-range palette index panics. Remote DoS via crafted upload.
- **Fix:** Upgrade `golang.org/x/image` to **v0.41.0+**. Add `defer recover()` around `image.Decode()` in upload handlers as a defense-in-depth measure.
- **Defense-in-depth pattern:**
  ```go
  func decodeImageSafely(r io.Reader) (image.Image, string, error) {
      var img image.Image
      var format string
      var err error

      func() {
          defer func() {
              if rec := recover(); rec != nil {
                  err = fmt.Errorf("image decode panic: %v", rec)
              }
          }()
          img, format, err = image.Decode(r)
      }()

      return img, format, err
  }
  ```

### CVE-2026-46602 / GO-2026-5062 — golang.org/x/image/tiff unbounded tile memory
- **Published:** 2026-06-25 (CVE published 2026-06-25) | **Severity:** Medium (no CVSS yet; CWE-789 memory allocation with excessive size value)
- **Affected:** `golang.org/x/image/tiff` versions **< v0.43.0**. Fix is in v0.43.0 (released 2026-06-15 — fix predated public CVE by 10 days).
- **Impact:** The TIFF decoder does not impose a limit on tile size in tiled images. A maliciously crafted TIFF can claim an excessively large tile dimension in the header, causing the decoder to allocate up to 4 GiB (or more) of memory per tile. Unauthenticated remote DoS against any Gin handler that calls `tiff.Decode` / `tiff.DecodeConfig` / `image.Decode` (which falls back to `tiff` if registered). Pattern is identical to the older CVE-2026-33809 / CVE-2022-41727 family — unbounded-size resource consumption via header-mutable inputs.
- **Gin exposure paths:** image-upload endpoints that whitelist TIFF (medical imaging, scanner/PDF ingestion, scientific data, screenshot uploads), any service that registers `tiff.Decode` with `image.RegisterFormat` for auto-detection, OCR pipelines, document conversion services.
- **Fix:** Upgrade `golang.org/x/image` to **v0.43.0+** (the skill's prior floor of v0.41.0 is **insufficient** — v0.41.0 fixed CVE-2026-42500 BMP panic, not this TIFF tile CVE). Run `go get golang.org/x/image@v0.43.0` and `go mod tidy`.
- **Defense-in-depth:** Same patterns as CVE-2026-42500 — (1) reject TIFF at the MIME sniff level if your domain doesn't need it (`"image/tiff"` is NOT in the standard `http.DetectContentType` whitelist, but `image.Decode` will still try TIFF if registered), (2) wrap `image.Decode` in `defer recover()` (see CVE-2026-42500 section), (3) cap decoded image dimensions with `image.DecodeConfig` before allocating a full decode buffer (`cfg.Width * cfg.Height * 4` byte ceiling — reject if > a sane limit like 50M pixels).

### CVE-2026-46604 / GO-2026-5066 — golang.org/x/image/tiff out-of-bounds strip offset panic (second x/image/tiff CVE in 48h)
- **Published:** 2026-06-26 20:04 UTC (NVD entry CVE-2026-46604 published 2026-06-26 21:16 UTC) | **Severity:** Medium (process panic / unauthenticated remote DoS; no CVSS yet, **CWE-787** out-of-bounds read → index out of range panic)
- **Affected:** `golang.org/x/image/tiff` versions **< v0.43.0**. Fix is in v0.43.0 (same release that fixed CVE-2026-46602 two days earlier — both bugs shipped in the same v0.43.0 release, but were tracked as separate CVEs).
- **Impact:** The TIFF decoder reads strip offsets from the IFD without bounds checking. A crafted TIFF can declare a strip offset that points past the end of the underlying byte stream; the decoder indexes into the raw buffer with the un-bounded offset and panics with `runtime error: index out of range`. Unauthenticated remote DoS against any Gin handler that calls `tiff.Decode` / `tiff.DecodeConfig` / `image.Decode` with TIFF registered. **Distinct from CVE-2026-46602** (which was unbounded tile-size memory allocation — that one allocates gigabytes before OOM; this one panics on the first malformed strip read).
- **Gin exposure paths:** identical to CVE-2026-46602 — image-upload endpoints that whitelist TIFF (medical imaging, scanner/PDF ingestion, scientific data, screenshot uploads), any service that registers `tiff.Decode` with `image.RegisterFormat` for auto-detection, OCR pipelines, document conversion services. **Both CVEs can be triggered by the same upload endpoint**, so defense-in-depth must cover both attack patterns (panic AND OOM).
- **Fix:** Upgrade `golang.org/x/image` to **v0.43.0+** (same floor as CVE-2026-46602 — already documented as the skill's `golang.org/x/image` floor). No new action required if you already applied the CVE-2026-46602 fix; `go get golang.org/x/image@v0.43.0 && go mod tidy` covers both.
- **Defense-in-depth:** Inherits all CVE-2026-46602 mitigations. Specifically for this panic variant: (1) the `defer recover()` wrap around `image.Decode` is **more critical** here than for the BMP panic (CVE-2026-42500) because this one fires on the first IFD read, not on a complex decode path — it's the first thing the decoder does, so the recover must wrap the entire `image.Decode` call, not a sub-function; (2) a pre-decode `http.MaxBytesReader` cap (e.g. 10 MiB) reduces the attacker-controlled buffer size, but does NOT prevent the panic — the offset is parsed from the header and can exceed even a large buffer, so this is **defense-in-breadth not in depth**; (3) prefer `image.DecodeConfig` first to validate header plausibility (check `cfg.Width > 0 && cfg.Height > 0 && cfg.Width*cfg.Height < saneLimit`) before allowing full decode.

### CVE-2026-39822 — Go runtime security fix (embargoed, imminent)
- **Status:** In release pipeline for Go 1.25.12 / 1.26.5 (verified 2026-06-21 12:04 UTC via dev.golang.org/release)
- **Affected:** Go runtime; details under embargo (issue #79005 / #79026 / #79027 marked private per Go security policy)
- **Action:** Track [golang-announce](https://groups.google.com/g/golang-announce) for the announcement thread; pre-stage builds with the patch within days of release.

### CVE-2026-42505 — Go `crypto/tls` PSK in ECH outer client hello (public-track privacy degradation)
- **Published:** 2026-05-08 (issue #79282 public-track per Go Security Policy); fix backports opened 2026-06-26 (#80174 for Go 1.25, #80175 for Go 1.26). **Severity:** Low–Medium (privacy degradation; no RCE/auth-bypass).
- **Affected:** Go `crypto/tls` client when negotiating TLS 1.3 with **Encrypted Client Hello (ECH)** outer ClientHello and a configured PSK. Affects any Gin service whose outbound HTTPS client uses `http.Transport` with a `tls.Config` that sets both `tls.Config.ECHConfigs` (or relies on ECH negotiation) and `tls.Config.PSK` / `tls.Config.PSKIdentity`.
- **Impact:** Including the PSK extension in the *outer* ClientHello allows an on-path attacker to harvest outer ClientHellos and replay them with arbitrary guessed SNI values. If the server accepts the PSK, the binder check fails (handshake errors); the privacy degradation is the fingerprintable SNI+PSK combination, not a credential leak. PSK is not leaked in the clear.
- **Fix:** Patched in **Go 1.25.12** (via #80174) and **Go 1.26.5** (via #80175). When these releases ship, the Go TLS client omits the PSK extension from the ECH outer ClientHello by default; only the inner ECH ClientHello carries the PSK.
- **Status (2026-07-01 18:24 UTC):** CLs #79282 (master), #80174 (1.25 backport), #80175 (1.26 backport) all **CLOSED in past 7 days** (now in "Closed Last Week" bucket on dev.golang.org/release). Patch release **imminent** (binaries NOT YET published to go.dev/dl — typical pre-release window 12-48h). CVE record still embargoed: `CVE_RECORD_DNE` on MITRE, `totalResults: 0` on NVD, not on pkg.go.dev/vuln/list. Go Security Team holding for coordinated publication with Go 1.25.12 / 1.26.5.
- **Gin exposure paths:** (1) any outbound `http.Client` from a Gin handler that uses a pre-configured `tls.Config` with PSK + ECH (rare but used in zero-trust mTLS / mesh sidecars); (2) Go HTTP clients following redirects to HTTPS endpoints that negotiate ECH; (3) any service that does **not** set PSK but relies on `crypto/tls` defaults is **NOT** affected — the bug only triggers when the caller explicitly configured a PSK.
- **Workaround for current Go (1.25.11 / 1.26.4 and earlier):** if your Gin service uses `tls.Config.PSK`, disable ECH for outbound calls (`tls.Config.ECHConfigs = nil` and do not set `EncryptedClientHelloConfigList` via QUIC/HTTP3). This is acceptable in nearly all Gin deployments because ECH + PSK is a niche combination. Track [golang-announce](https://groups.google.com/g/golang-announce) for the official advisory thread once Go 1.25.12 / 1.26.5 ship.
- **Related:** issue [#79282](https://github.com/golang/go/issues/79282) (public-track; details above), backports [#80174](https://github.com/golang/go/issues/80174) (Go 1.25) and [#80175](https://github.com/golang/go/issues/80175) (Go 1.26). NVD record not yet populated as of 2026-06-26 (CVE reserved).

### CVE-2026-56853 — Go `net/http` UnencryptedHTTP2 read-header-timeout bypass (DoS amplifier, public-track)
- **Published:** 2026-07-01 (issue #80205 opened 2026-07-01T16:15:10Z as public-track per Go Security Policy; **NOT YET in NVD** as of 2026-07-02 06:14 UTC — `totalResults: 0` — MITRE also `CVE_RECORD_DNE`). **Severity:** Medium (slow-loris-style resource exhaustion on h2c servers; no RCE/auth-bypass).
- **Affected:** Go `net/http` server when configured with **both** `http.Server.UnencryptedHTTP2 = true` (h2c enabled) **and** `http.Server.ReadHeaderTimeout` set. The h2c client-preface sniff reads bytes from the connection without applying `ReadHeaderTimeout`, so idle connections linger without timeout until the OS-level TCP timeout expires.
- **Impact:** Slow-loris-style DoS amplifier. An attacker can hold many connections open with garbage bytes and accumulate them up to TCP-timeout limits (typically minutes to hours), amplifying memory and file-descriptor consumption by 10⁴–10⁶× per connection depending on OS TCP tuning. Process memory and FD table grow until exhaustion or OOM-kill. **CWE-400** (uncontrolled resource consumption).
- **Gin exposure paths:** (1) any Gin service that explicitly opts into h2c via `gin.Engine.RunH2C()` or by setting `http.Server.UnencryptedHTTP2 = true` AND sets `ReadHeaderTimeout` (the combination that triggers the bug — **rare in production** because h2c is typically only used in dev/internal contexts); (2) any Gin service using `gin.Engine.RunH2C()` with custom `http.Server` configured separately for `ReadHeaderTimeout` (would need manual wiring — `gin.RunH2C` does NOT set ReadHeaderTimeout by default, so default deployments are NOT affected); (3) any service using Go's `golang.org/x/net/http2` server in cleartext mode. **Standard TLS-terminated Gin services (`gin.Engine.RunTLS` or behind a reverse proxy) are NOT affected.**
- **Fix:** Patched in **Go 1.25.13 / 1.26.6 / Go 1.27-rc3**. Master commit 1952e61 was landed but **TEMPORARILY REVERTED** on 2026-07-01T18:17:18Z (per `neild`: "landed too close to the rc2 cutoff") — will appear in the **next** set of minor releases after 1.25.12 / 1.26.5. Backport tracking issues: [#80223](https://github.com/golang/go/issues/80223) (Go 1.25) and [#80224](https://github.com/golang/go/issues/80224) (Go 1.26), both opened by gopherbot on 2026-07-01T16:15:10Z with `[NeedsFix, CherryPickApproved]` labels.
- **Status (2026-07-02 06:14 UTC):** Backport tracking issues open; cherry-pick CLs NOT YET committed. Master commit reverted; will re-land in rc3 window (~2026-07-09 cadence or later). CVE record still unpublished (`CVE_RECORD_DNE` on MITRE, `totalResults: 0` on NVD, not on pkg.go.dev/vuln/list).
- **Gin exposure mitigation for current Go (1.25.11 / 1.26.4 and earlier):**
  1. **Disable h2c if not needed**: `http.Server.UnencryptedHTTP2 = false` (Gin default — `gin.RunH2C` is the only built-in way to enable it). Most Gin services do not need h2c.
  2. **If h2c is required**: set `http.Server.IdleTimeout` to a low value (e.g. 10–30 seconds) to bound connection lifetime independently of `ReadHeaderTimeout`.
  3. **Defense-in-depth**: front the service with a reverse proxy that enforces connection timeouts at the edge (nginx `proxy_read_timeout`, envoy `connection_timeout`).
- **Related:** issue [#80205](https://github.com/golang/go/issues/80205) (master public-track; credits Vsevolod Naumov and Ainar Garipov from AdGuard for reporting), backport tracking issues [#80223](https://github.com/golang/go/issues/80223) (Go 1.25) and [#80224](https://github.com/golang/go/issues/80224) (Go 1.26), master commit [1952e61](https://github.com/golang/go/commit/1952e61) (reverted, will re-land in rc3). Reporter credits per [neild 2026-06-30T22:33:25Z comment](https://github.com/golang/go/issues/80205#issuecomment-3106665028).

### CVE-2026-39821 / GO-2026-5026 — golang.org/x/net idna Punycode privilege escalation (CRITICAL, CVSS 10.0)
- **Published:** 2026-05-22 | **Severity:** Critical
- **Affected:** `golang.org/x/net` versions < v0.55.0 (idna package — `ToASCII` / `ToUnicode`)
- **Impact:** The idna package's `ToASCII` / `ToUnicode` incorrectly accept Punycode-encoded labels that decode to ASCII-only labels. Example: `ToUnicode("xn--example-.com")` incorrectly returns `"example.com"` instead of an error. **Enables privilege escalation** — a handler that performs allow/deny checks on the ASCII hostname (e.g. validating OAuth `redirect_uri` host against an allowlist) may reject `"example.com"` but **permit `"xn--example-.com"`**, which then normalizes back to `"example.com"` downstream. Bypasses host-based authorization checks.
- **Gin exposure paths:** webhook receiver handlers that validate `Host` header, OAuth/OIDC `redirect_uri` validators, allowlist middleware for SSO callbacks, anything using `c.Request.Host` plus idna.
- **Fix:** Upgrade to `golang.org/x/net` **v0.55.0+**. Pin `golang.org/x/net/idna` directly if you only use that subpackage.
- **Defense-in-depth:** Normalize hostnames with `idna.Lookup.ToASCII` (not raw punycode passthrough) before allowlist comparison.

### CVE-2026-25680 / GO-2026-5028 — golang.org/x/net HTML parser CPU DoS
- **Published:** 2026-05-22 | **Severity:** Medium
- **Affected:** `golang.org/x/net` < v0.55.0 (html package)
- **Impact:** Parsing arbitrary, deeply-nested or pathological HTML can consume excessive CPU time — remote DoS via crafted input.
- **Gin exposure paths:** any handler that accepts HTML body input and calls `html.Parse` / `golang.org/x/net/html` for sanitization (e.g. rich-text editors, email preview, pastebins).
- **Fix:** Upgrade to `golang.org/x/net` **v0.55.0+**. Add a request body size limit (already recommended for any user-uploaded HTML).
- **Workaround:** wrap `html.Parse` with a `context.Context` timeout via goroutine + select pattern, and cap input size before parsing.

### CVE-2026-42502 / GO-2026-5027 — golang.org/x/net html.Render XSS re-introduction
- **Published:** 2026-05-22 | **Severity:** Medium
- **Affected:** `golang.org/x/net` < v0.55.0 (html package — `Render` function)
- **Impact:** Parsing arbitrary HTML which is then rendered using `Render` can produce an unexpected HTML tree that **re-introduces XSS even after input sanitization**. Applications that try to sanitize HTML before rendering can be bypassed.
- **Gin exposure paths:** any handler using `html.Render` to display user-supplied HTML (markdown preview, comment rendering, email body).
- **Fix:** Upgrade to `golang.org/x/net` **v0.55.0+**. For new projects, prefer `bluemonday` or `html-sanitizer` for HTML sanitization — never rely on `html.Render` for XSS prevention.

### CVE-2026-39824 / GO-2026-5024 — golang.org/x/sys windows NTUnicodeString overflow
- **Published:** 2026-05-22 | **Severity:** Medium (Windows-only)
- **Affected:** `golang.org/x/sys/windows` < v0.46.0 (Debian backport already in sid/forky/trixie; upstream fix in v0.46.0)
- **Impact:** `NewNTUnicodeString` does not check for string length overflow. When given a string longer than the 16-bit NTUnicodeString max, it **silently returns a truncated string** instead of an error. This can lead to incorrect path/registry operations on Windows.
- **Gin exposure paths:** Windows-only — affects Gin apps using `golang.org/x/sys/windows` for native Windows API calls (rare in container deployments; relevant for Windows service or IIS-hosted Gin apps).
- **Fix:** Upgrade to `golang.org/x/sys` **v0.46.0+** (already bundled with Go 1.26 toolchain).
- **Mitigation:** After calling `NewNTUnicodeString`, validate `len(result) == len(input)` or check for nil/empty result when input was non-empty.

### CVE-2026-46598 / GO-2026-5033 — golang.org/x/crypto ssh/agent ed25519 panic
- **Published:** 2026-05-22 | **Severity:** Medium
- **Affected:** `golang.org/x/crypto` < v0.52.0 (ssh/agent package — ed25519 wire-format parsing)
- **Impact:** Malformed ed25519 wire bytes cast into `ed25519.PrivateKey` cause a **panic when used**. Unauthenticated remote attacker can crash any Gin service that imports `golang.org/x/crypto/ssh/agent` (or its transitive importers).
- **Gin exposure paths:** rare for HTTP servers — but any service using `ssh-agent` forwarding, jump-host proxies, or build-time SSH key parsing is affected. **Crypto/SSH-facing admin APIs are most at risk.**
- **Fix:** Upgrade to `golang.org/x/crypto` **v0.53.0+** (v0.52.0 was the initial patch release on 2026-05-22; v0.53.0 is current). Add `defer recover()` around `ed25519.PrivateKey` decoding as defense-in-depth.

### CVE-2026-46597 / GO-2026-5013 — golang.org/x/crypto ssh AES-GCM panic
- **Published:** 2026-05-22 | **Severity:** Medium
- **Affected:** `golang.org/x/crypto` < v0.52.0 (ssh package — AES-GCM packet decoder)
- **Impact:** Incorrectly-placed `bytes→int` cast in AES-GCM packet decoder causes **server-side panic on well-crafted inputs**. Unauthenticated remote DoS against any SSH server using `golang.org/x/crypto/ssh`.
- **Gin exposure paths:** Gin apps embedding an SSH server or admin shell (rare but used for ops endpoints). Transitive risk: `crypto/ssh` is imported by many ops tools.
- **Fix:** Upgrade to `golang.org/x/crypto` **v0.53.0+**.

### CVE-2026-39828, CVE-2026-39835, CVE-2026-39827, CVE-2026-39830, CVE-2026-39831, CVE-2026-39829 — golang.org/x/crypto ssh server hardening batch
- **Published:** 2026-05-22 | **Severity:** Mixed (Medium-High)
- **Affected:** `golang.org/x/crypto` < v0.52.0 (ssh package — server callbacks, certificate handling, agent key constraints)
- **Highlights (full list in [golang-announce thread](https://groups.google.com/g/golang-announce/c/a082jnz-LvI)):**
  - **CVE-2026-39828** — ssh certificate restriction bypass: when SSH server auth callback returns `PartialSuccessError` with non-nil `Permissions`, those permissions were silently discarded, dropping certificate restrictions (e.g. `force-command`) after a second factor succeeded. Now permissions are honored.
  - **CVE-2026-39835** — ssh server panic during `CheckHostKey`/`Authenticate`.
  - **CVE-2026-39827** — ssh/agent key constraints not enforced on key use.
  - **CVE-2026-39830** — ssh client deadlock on unexpected server responses.
  - **CVE-2026-39831** — ssh server DoS via pathological RSA/DSA parameters.
  - **CVE-2026-39829** — additional ssh hardening fix.
- **Gin exposure paths:** any service that ships an embedded SSH admin server / bastion endpoint using `crypto/ssh`.
- **Fix:** Upgrade to `golang.org/x/crypto` **v0.53.0+** — all six CVEs are fixed in v0.52.0 / v0.53.0.

### CVE-2026-46595 / GO-2026-5023 — golang.org/x/crypto ssh callback source-address authz bypass
- **Published:** 2026-05-22 | **Severity:** High
- **Affected:** `golang.org/x/crypto` < v0.52.0 (ssh server — callback handling)
- **Impact:** Partial fix for CVE-2024-45337 — if a non-public-key callback type was passed, **source-address validation was skipped**, allowing auth bypass when SSH servers are configured with multi-factor callbacks.
- **Gin exposure paths:** embedded SSH admin endpoints with multi-factor callbacks.
- **Fix:** Upgrade to `golang.org/x/crypto` **v0.53.0+**.

### CVE-2026-42508 / GO-2026-5021 — golang.org/x/crypto SSH CA SignatureKey revocation check
- **Published:** 2026-05-22 | **Severity:** Medium
- **Affected:** `golang.org/x/crypto` < v0.52.0 (ssh server — CA cert handling)
- **Impact:** A revoked CA `SignatureKey` was not correctly checked for revocation during SSH certificate validation. Now both `key` and `key.SignatureKey` are checked for `@revoked`. SSH clients/servers trusting SSH CA certs could accept certificates signed by a compromised/revoked CA key.
- **Gin exposure paths:** any embedded SSH endpoint that validates user certificates against an SSH CA (common for bastion/jump-host patterns).
- **Fix:** Upgrade to `golang.org/x/crypto` **v0.53.0+**.

### CVE-2026-22786 — Gin-vue-admin path traversal (applies to Gin handlers generally)
- **Published:** 2026-01-12 | **Severity:** High
- **Issue:** Concatenating user-supplied filenames with directory paths using `os.OpenFile()` without sanitization allows `../` traversal and arbitrary file writes.
- **Lesson for raw Gin handlers:** Always sanitize filenames with `filepath.Base()` and reject empty results:
  ```go
  import "path/filepath"

  func safeFilename(userInput string) (string, error) {
      base := filepath.Base(userInput)
      if base == "." || base == "/" || base == "" || strings.Contains(base, "..") {
          return "", fmt.Errorf("invalid filename")
      }
      // Optional: further restrict to a character class
      matched, _ := regexp.MatchString(`^[a-zA-Z0-9._-]+$`, base)
      if !matched {
          return "", fmt.Errorf("filename contains disallowed characters")
      }
      return base, nil
  }
  ```

### PR #4726 — Gin `cleanPath` scheme-relative / backslash redirect (OPEN, not yet merged)
- **Opened:** 2026-07-02 23:26 UTC by NiiMER | **Updated:** 2026-07-02 23:57 UTC | **Status:** OPEN, no milestone, no review comments yet (only Codecov report @ 50% coverage on the patch lines).
- **File changed:** `path.go` (cleanPath function) — +7 lines, -0 lines.
- **The fix (verbatim from PR diff):**
  ```go
  // Prevent scheme-relative ("//...") or backslash-based absolute ("/\\...") paths.
  for len(p) > 1 && p[0] == '/' && p[1] == '/' {
      p = p[1:]
  }
  if len(p) > 1 && p[0] == '/' && p[1] == '\\' {
      p = "/" + p[2:]
  }
  ```
- **Impact:** `cleanPath` is invoked by Gin's router on every incoming request. Before this patch, an incoming URL like `//evil.com/foo` or `/\\evil.com/foo` is treated as a same-origin path (the second `/` or `\\` is collapsed during path normalization). A handler that calls `c.Redirect(http.StatusFound, cleanedPath)` then issues a `Location` header pointing at the attacker-controlled host — classic open-redirect / SSRF vector when the redirect target is later fetched server-side (OAuth callbacks, post-logout redirects, link unfurlers, webhook testers, email "click here" links).
- **Severity:** Medium–High (open-redirect class; severity escalates to High if your handler follows the redirect server-side or uses the path as a `url.URL.Host` source).
- **Gin exposure paths:** (1) any handler that uses `c.Request.URL.Path` (post-cleanPath) as a redirect target without re-validating the host; (2) OAuth/OIDC `post_logout_redirect_uri` or `redirect_uri` validators that pass through path-only strings; (3) "share link" / "deep link" handlers that redirect to user-supplied paths; (4) reverse-proxy frontends that forward the cleaned path as an upstream redirect. **Static-file and JSON API endpoints are NOT affected** because they do not issue redirects.
- **Mitigation for current Gin (v1.12.0 and earlier) — apply until PR #4726 lands:**
  ```go
  // In any handler that does c.Redirect with a path derived from user input:
  func safeRedirectPath(raw string) (string, error) {
      // 1. Reject scheme-relative and backslash-based paths outright.
      if len(raw) >= 2 && raw[0] == '/' && (raw[1] == '/' || raw[1] == '\\') {
          return "", fmt.Errorf("invalid redirect path: scheme-relative or backslash form")
      }
      // 2. Reject anything that's not an absolute path on our origin.
      if !strings.HasPrefix(raw, "/") {
          return "", fmt.Errorf("redirect path must be absolute")
      }
      // 3. Defense-in-depth: parse and re-check host.
      u, err := url.Parse(raw)
      if err != nil || u.Host != "" {
          return "", fmt.Errorf("redirect path must not carry a host")
      }
      return raw, nil
  }
  ```
- **Track:** [PR #4726](https://github.com/gin-gonic/gin/pull/4726) — once merged, the upstream `cleanPath` will reject these inputs at the framework level and this mitigation becomes redundant (but harmless).

## Production Server Defaults Cheat Sheet

| Timeout              | Recommended | Purpose                                           |
|----------------------|-------------|---------------------------------------------------|
| `ReadHeaderTimeout`  | 5–10s       | Bounds slow-header / slowloris DoS                |
| `ReadTimeout`        | 30s         | Total request body read time                      |
| `WriteTimeout`       | 30s         | Response write deadline                           |
| `IdleTimeout`        | 120s        | Keep-alive idle (after Go 1.8)                    |
| `MaxHeaderBytes`     | 1 MB        | Cap memory per request for headers                |
| `MaxMultipartMemory` | 8–32 MB     | Gin-specific; cap per-handler multipart memory    |

## Cross-Origin Isolation (COOP/COEP)

Required when you need `SharedArrayBuffer` (e.g., WebAssembly threads, high-resolution timers):

```go
func CrossOriginIsolation() gin.HandlerFunc {
    return func(c *gin.Context) {
        // COEP: Require embedded resources to grant explicit permission
        // Enables cross-origin isolation — needed for SharedArrayBuffer, performance.measureUserAgentSpecificMemory()
        c.Header("Cross-Origin-Embedder-Policy", "require-corp")
        // COOP: Control how cross-origin windows interact with your window
        c.Header("Cross-Origin-Opener-Policy", "same-origin")
        c.Next()
    }
}
```

**When to use COOP/COEP:**
- SharedArrayBuffer (WASM threads, high-res `performance.now()`)
- `performance.measureUserAgentSpecificMemory()`
- `navigator.storage.estimate()` with detailed info

**When NOT to use** — breaks embedding of third-party content that doesn't allow being framed:
- Third-party widgets (YouTube, Twitter embeds, etc.)
- OAuth popups to cross-origin identity providers
- `Cross-Origin-Opener-Policy: same-origin` blocks all cross-origin window interactions

**Tip:** Test with `self.crossOriginIsolated` in browser console — returns `true` when both headers are set correctly.

## Content-Security-Policy with Reporting (Modern Reporting API)

`report-uri` is deprecated — use `report-to` with the [Reporting API](https://developer.mozilla.org/en-US/docs/Web/API/Reporting_API) instead:

```go
func CSPWithReporting() gin.HandlerFunc {
    return func(c *gin.Context) {
        // 1. Register the reporting endpoint via Report-To header
        c.Header("Report-To", `{"group":"csp-endpoint","max_age":86400,"endpoints":[{"url":"/csp-report"}]}`)
        // 2. Use report-to in CSP (not the deprecated report-uri)
        c.Header("Content-Security-Policy",
            "default-src 'self'; report-to csp-endpoint; style-src 'self' 'unsafe-inline'")
        c.Next()
    }
}

// Modern CSP violation handler — uses the Reporting API format
func cspReportHandler(c *gin.Context) {
    var reports []struct {
        Type        string `json:"type"`
        URL         string `json:"url"`
        Age         int    `json:"age"`
        Status      int    `json:"status"`
        Body        struct {
            Directive string `json:"directive"`
            Policy    string `json:"policy"`
            SourceFile string `json:"sourceFile"`
            LineNumber int    `json:"lineNumber"`
            ColumnNumber int `json:"columnNumber"`
        } `json:"body"`
    }

    // The Reporting API sends reports as JSON array, not the old csp-report wrapper
    if err := c.ShouldBindJSON(&reports); err != nil {
        c.JSON(400, gin.H{"error": "invalid report"})
        return
    }

    for _, report := range reports {
        log.Printf("CSP violation [%s]: %s — %s (line %d)",
            report.Type,
            report.URL,
            report.Body.Directive,
            report.Body.LineNumber)
    }

    c.Status(204)
}
```

**Key differences from deprecated `report-uri`:**
- `report-to` uses the modern [Reporting API](https://developer.mozilla.org/en-US/docs/Web/API/Reporting_API) — reports are sent as `Content-Type: application/reports+json`
- Reports include `type`, `url`, `age`, `status`, and `body` — richer information than the old `csp-report` wrapper
- `max_age` controls how long the endpoint is remembered by the browser

## security.txt (RFC 9116)

```go
// /.well-known/security.txt endpoint
func securityTxtHandler(c *gin.Context) {
    c.Header("Content-Type", "text/plain; charset=utf-8")
    c.String(200, `Contact: security@example.com
Encryption: https://example.com/pgp-key.txt
Preferred-Languages: en
Canonical: https://api.example.com/.well-known/security.txt
Policy: https://example.com/security-policy.html
`)
}
```

## Input Validation

```go
import "regexp"

// Always validate, never trust input
func validateEmail(email string) bool {
    re := regexp.MustCompile(`^[a-z0-9._%+-]+@[a-z0-9.-]+\.[a-z]{2,}$`)
    return re.MatchString(email)
}

// Or use binding tags
type CreateUserRequest struct {
    Name     string `json:"name" binding:"required,min=2,max=50"`
    Email    string `json:"email" binding:"required,email"`
    Password string `json:"password" binding:"required,min=8"`
    Age      int    `json:"age" binding:"min=18,max=120"`
}

// Custom validator
import "github.com/go-playground/validator/v10"

func RegisterCustomValidators(v *validator.Validate) {
    v.RegisterValidation("strong-password", func(fl validator.FieldLevel) bool {
        password := fl.Field().String()
        if len(password) < 8 {
            return false
        }
        hasUpper := false
        hasLower := false
        hasDigit := false
        for _, c := range password {
            switch {
            case c >= 'A' && c <= 'Z':
                hasUpper = true
            case c >= 'a' && c <= 'z':
                hasLower = true
            case c >= '0' && c <= '9':
                hasDigit = true
            }
        }
        return hasUpper && hasLower && hasDigit
    })
}
```

## SQL Injection Prevention

```go
// WRONG — never do this
query := fmt.Sprintf("SELECT * FROM users WHERE email = '%s'", email)
row := DB.Raw(query)

// RIGHT — parameterized queries
row := DB.Raw("SELECT * FROM users WHERE email = ?", email)

// With GORM (automatically uses parameterized queries)
var user User
DB.Where("email = ?", email).First(&user)
```

## XSS Prevention

```go
import "html/template"

// When rendering user content, escape HTML
func renderUserContent(c *gin.Context, content string) {
    // template.HTMLEscapeString escapes < > & " '
    safe := template.HTMLEscapeString(content)
    c.String(200, safe)
}

// Or use template functions in HTML responses
c.HTML(200, "user_post.html", gin.H{
    "content": template.HTMLEscape(template.HTML(user.Content)), // only after sanitization
})
```

## CORS Configuration

```go
import "github.com/gin-contrib/cors"

r := gin.New()

// Simple config
r.Use(cors.New(cors.Config{
    AllowOrigins:     []string{"https://myapp.com"},
    AllowMethods:     []string{"GET", "POST", "PUT", "DELETE", "OPTIONS"},
    AllowHeaders:     []string{"Origin", "Content-Type", "Authorization"},
    ExposeHeaders:    []string{"X-Request-ID"},
    AllowCredentials: true,
    MaxAge:           12 * time.Hour,
}))
```

## Rate Limiting

```go
import (
    "net/http"
    "golang.org/x/time/rate"
)

type RateLimiter struct {
    limiter  *rate.Limiter
    requests int
}

func PerClientRateLimit(requestsPerSecond float64, burst int) gin.HandlerFunc {
    limiters := &sync.Map{}
    
    return func(c *gin.Context) {
        clientIP := c.ClientIP()
        
        limiterInterface, _ := limiters.LoadOrStore(clientIP, 
            rate.NewLimiter(rate.Limit(requestsPerSecond), burst))
        limiter := limiterInterface.(*rate.Limiter)
        
        if !limiter.Allow() {
            c.AbortWithStatusJSON(http.StatusTooManyRequests, gin.H{
                "error": "rate limit exceeded",
                "retry_after": "60",
            })
            return
        }
        
        c.Next()
    }
}

r.Use(PerClientRateLimit(10.0, 20)) // 10 req/s, burst 20
```

### Redis-backed Distributed Rate Limiting

```go
// For multi-instance deployments, use Redis for rate limiting
import "github.com/redis/go-redis/v9"

func RedisRateLimit(rps float64, burst int) gin.HandlerFunc {
    return func(c *gin.Context) {
        ctx := c.Request.Context()
        key := "ratelimit:" + c.ClientIP()
        
        // Sliding window counter
        now := time.Now().Unix()
        windowKey := fmt.Sprintf("%s:%d", key, now)
        
        count, err := redisClient.Incr(ctx, windowKey).Result()
        if err != nil {
            c.Next() // fail open if Redis is down
            return
        }
        
        // Set expiry on first request in window
        if count == 1 {
            redisClient.Expire(ctx, windowKey, time.Second)
        }
        
        // Allow burst + rps requests per second
        limit := int64(float64(burst) + rps)
        if count > limit {
            c.Header("X-RateLimit-Limit", fmt.Sprintf("%d", int64(rps)))
            c.Header("X-RateLimit-Remaining", "0")
            c.AbortWithStatusJSON(http.StatusTooManyRequests, gin.H{
                "error": "rate limit exceeded",
                "retry_after": "1",
            })
            return
        }
        
        c.Next()
    }
}
```

## Request Size Limiting

```go
// Limit request body size (10MB)
r.Use(func(c *gin.Context) {
    if c.Request.ContentLength > 10<<20 {
        c.AbortWithStatusJSON(413, gin.H{"error": "request too large"})
        return
    }
    
    // Limit memory used for body parsing
    c.Request.Body = http.MaxBytesReader(c.Writer, c.Request.Body, 10<<20)
    c.Next()
})
```

## HTTPS Enforcement

```go
func RequireHTTPS() gin.HandlerFunc {
    return func(c *gin.Context) {
        if c.Request.TLS == nil && getEnv("ENV", "dev") == "production" {
            c.AbortWithStatusJSON(400, gin.H{"error": "HTTPS required"})
            return
        }
        c.Next()
    }
}
```

## Secret Management

```go
import "os"

// Never hardcode secrets
var jwtSecret = []byte(os.Getenv("JWT_SECRET"))
var dbPassword = os.Getenv("DB_PASSWORD")

// Validate required env vars at startup
func loadEnvVars() error {
    required := []string{"DATABASE_URL", "JWT_SECRET", "REDIS_URL"}
    for _, key := range required {
        if os.Getenv(key) == "" {
            return fmt.Errorf("required env var %s not set", key)
        }
    }
    return nil
}
```

## CSRF Protection

```go
// For browser-based applications, protect against CSRF
import "github.com/gin-contrib/csrf"

func CSRFSetup() gin.HandlerFunc {
    return csrf.New(csrf.Options{
        Secret:     os.Getenv("CSRF_SECRET"),
        CookieName: "_csrf",
        CookiePath: "/",
        Secure:     getEnv("ENV", "dev") == "production",
        SameSite:   http.SameSiteLaxMode,
    })
}
```

## OAuth2 PKCE (Public Clients — Mobile Apps, SPAs)

PKCE (Proof Key for Code Exchange) prevents authorization code interception attacks for public clients.

```go
import (
    "crypto/rand"
    "crypto/sha256"
    "encoding/base64"
    "encoding/json"
)

// Generate PKCE code verifier and challenge
func GeneratePKCE() (verifier string, challenge string, err error) {
    verifierBytes := make([]byte, 32)
    if _, err := rand.Read(verifierBytes); err != nil {
        return "", "", err
    }
    verifier = base64.RawURLEncoding.EncodeToString(verifierBytes)
    
    // SHA256 hash of verifier, base64url encoded (no padding)
    hash := sha256.Sum256([]byte(verifier))
    challenge = base64.RawURLEncoding.EncodeToString(hash[:])
    
    return verifier, challenge, nil
}

// OAuth2 authorization URL with PKCE
func BuildAuthorizationURL(state string, pkceVerifier string) string {
    // challenge is already generated: Base64(SHA256(verifier))
    // Include in authorization request
    // On your backend:
    codeChallenge := generateCodeChallenge(pkceVerifier) // "E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbzGZpVLQhM2E"
    
    return "https://auth.example.com/authorize?" +
        "response_type=code" +
        "&client_id=YOUR_CLIENT_ID" +
        "&redirect_uri=https%3A%2F%2Fyourapp%2Fcallback" +
        "&scope=openid%20profile%20email" +
        "&state=" + state +
        "&code_challenge=" + codeChallenge +
        "&code_challenge_method=S256"
}

// Exchange authorization code with PKCE verifier
func ExchangeCodeForTokens(ctx context.Context, code string, pkceVerifier string) (*TokenResponse, error) {
    // Verify the code verifier matches the challenge
    // Server stores code_challenge when issuing code, verifies S256(verifier) == code_challenge
    
    params := url.Values{
        "grant_type":    {"authorization_code"},
        "code":          {code},
        "redirect_uri":  {"https://yourapp/callback"},
        "client_id":     {"YOUR_CLIENT_ID"},
        "code_verifier": {pkceVerifier}, // Must match original challenge
    }
    
    req, _ := http.NewRequestWithContext(ctx, "POST", "https://auth.example.com/token", 
        strings.NewReader(params.Encode()))
    req.Header.Set("Content-Type", "application/x-www-form-urlencoded")
    
    // ... execute request
}

func generateCodeChallenge(verifier string) string {
    h := sha256.Sum256([]byte(verifier))
    return base64.RawURLEncoding.EncodeToString(h[:])
}
```

**When to use PKCE:**
- Mobile apps (iOS, Android) — no client secret possible
- SPAs (React, Vue) — client secret in frontend is not secret
- Any OAuth2 flow where code could be intercepted

## Common Mistakes

1. **No security headers** — browsers won't protect against XSS/clickjacking
2. **Raw SQL with string formatting** — SQL injection vulnerability
3. **No rate limiting** — API gets hammered, DoS vulnerability
4. **Request body size not limited** — memory exhaustion attack vector
5. **Secrets in code** — environment variables only, never hardcode
6. **No CORS lock-down** — `*` origins in production exposes API
7. **Missing HSTS header** — without it, SSL stripping attacks are possible
8. **No CSRF for browser apps** — state-changing APIs need CSRF protection
9. **OAuth2 without PKCE on public clients** — authorization code interception risk
10. **No CSP reporting** — CSP violations go undetected
11. **Using deprecated `report-uri` in CSP** — switch to `report-to` (Reporting API)
12. **Missing COOP/COEP when SharedArrayBuffer is needed** — breaks WASM threading

---

## Updated from Research (2026-05)

### CSP Reporting API (Modern — replaces deprecated report-uri)
- `report-uri` directive is deprecated in favor of `report-to` and the [Reporting API](https://developer.mozilla.org/en-US/docs/Web/API/Reporting_API)
- Reports are sent as `Content-Type: application/reports+json` (array format), not `application/json` (single `csp-report` wrapper)
- `Report-To` header registers the endpoint with `max_age`, `group`, and `endpoints` array
- Use `max_age` to control how long the browser remembers the endpoint

### COOP/COEP (Cross-Origin Isolation)
- `Cross-Origin-Opener-Policy: same-origin` — your window can only communicate with windows from the same origin
- `Cross-Origin-Embedder-Policy: require-corp` — embedded resources must explicitly grant permission via `Cross-Origin-Resource-Policy`
- Required for `SharedArrayBuffer` (WASM threads), `performance.measureUserAgentSpecificMemory()`, detailed `navigator.storage.estimate()`
- Check `self.crossOriginIsolated` in browser console — `true` means both headers are active

### OAuth2 PKCE (RFC 7636)
- Required for public clients (mobile, SPAs) — no client secret can be kept secret in these environments
- Code verifier + challenge exchange prevents auth code interception attacks
- `code_challenge_method=S256` preferred over `plain`

### Redis-backed Distributed Rate Limiting
- For horizontally scaled Gin apps (multiple instances), in-memory rate limiting doesn't work — all instances need to share state via Redis
- Sliding window counter pattern using Redis `INCR` + `EXPIRE` is simple and effective

### CSRF Protection
- Stateless JWT-based APIs are generally not vulnerable to CSRF (no cookies), but browser-based apps with session cookies need CSRF tokens
- `github.com/gin-contrib/csrf` provides Gin middleware integration

### Sources
- OWASP Security Headers: https://owasp.org/www-project-secure-headers/
- HSTS Preload List: https://hstspreload.org/
- RFC 9116 (security.txt): https://www.rfc-editor.org/rfc/rfc9116
- RFC 7636 (PKCE): https://www.rfc-editor.org/rfc/rfc7636
- MDN Reporting API: https://developer.mozilla.org/en-US/docs/Web/API/Reporting_API
- COOP/COEP: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cross-Origin-Embedder-Policy

---

## Updated from Research (2026-06-22, 18:04 UTC)

### May 22, 2026 golang.org/x/crypto + x/net + x/sys security batch added

Added nine new CVE entries to the "Recent Gin/Go CVEs (May–June 2026)" section covering the Go security announcements published on 2026-05-22 (T-minus 31 days from today). The skill previously had no coverage of this batch — only the May/June CVE list in versions.md referenced individual CVEs without detail.

**New entries added (security.md):**

1. **CVE-2026-39821 / GO-2026-5026** — golang.org/x/net idna Punycode privilege escalation (**CRITICAL, CVSS 10.0**) — fixed in x/net v0.55.0. Hostname allowlist bypass via `xn--example-.com` → `example.com` normalization. Critical for OAuth redirect-URI validators, webhook host checks, SSO callback allowlists.
2. **CVE-2026-25680 / GO-2026-5028** — golang.org/x/net HTML parser CPU DoS — fixed in x/net v0.55.0.
3. **CVE-2026-42502 / GO-2026-5027** — golang.org/x/net `html.Render` XSS re-introduction — fixed in x/net v0.55.0. Affects Gin handlers displaying user HTML (markdown preview, comment rendering).
4. **CVE-2026-39824 / GO-2026-5024** — golang.org/x/sys/windows `NewNTUnicodeString` integer overflow (Windows-only) — fixed in x/sys v0.46.0 (already in Go 1.26 toolchain). Silently truncates oversized strings instead of erroring.
5. **CVE-2026-46598 / GO-2026-5033** — golang.org/x/crypto ssh/agent ed25519 wire-byte panic — fixed in x/crypto v0.52.0 (current: v0.53.0).
6. **CVE-2026-46597 / GO-2026-5013** — golang.org/x/crypto ssh AES-GCM packet decoder panic — fixed in x/crypto v0.52.0+.
7. **CVE-2026-39828, CVE-2026-39835, CVE-2026-39827, CVE-2026-39830, CVE-2026-39831, CVE-2026-39829** — golang.org/x/crypto ssh server hardening batch (6 CVEs) — certificate restrictions bypass, server panic, agent key constraints, client deadlock, RSA/DSA DoS, additional hardening.
8. **CVE-2026-46595 / GO-2026-5023** — golang.org/x/crypto ssh callback source-address authz bypass (High) — follow-up to CVE-2024-45337.
9. **CVE-2026-42508 / GO-2026-5021** — golang.org/x/crypto SSH CA `SignatureKey` revocation check — both `key` and `key.SignatureKey` now checked for `@revoked`.

### Section title updated
- "Recent Gin/Go CVEs (June 2026)" → "Recent Gin/Go CVEs (May–June 2026)" to reflect the broader date range now covered.

### Why this matters for Gin developers
- **Most Gin apps are NOT directly exposed** to the x/crypto and x/net SSH/HTML CVEs unless they embed an SSH admin endpoint or render user-supplied HTML.
- **HOWEVER:** `golang.org/x/net` is a near-universal transitive dependency (HTTP/2, HTML escaping in `template.HTMLEscapeString`, etc.). Go modules resolution means **v0.55.0 should be the minimum pin** to clear all May CVEs.
- The May 22 batch was released **31 days ago** — agents deploying new Gin services should add these version floors to their go.mod and verify with `govulncheck ./...` in CI.

### Detection
- Run `govulncheck ./...` against the Gin project to surface any of these CVEs in transitive deps
- Check `go.mod` for `golang.org/x/net < v0.55.0` and `golang.org/x/crypto < v0.53.0`
- For Windows deployments: `golang.org/x/sys < v0.46.0`

### Sources
- https://groups.google.com/g/golang-announce (golang-announce, 2026-05-22 batch)
  - x/crypto: https://groups.google.com/g/golang-announce/c/a082jnz-LvI
  - x/net/html: https://groups.google.com/g/golang-announce/c/iI-mYSI0lu8 (CVE-2026-39821)
  - x/sys: https://groups.google.com/g/golang-announce/c/6MMI8Lj-Atg (CVE-2026-39824)
- https://pkg.go.dev/vuln/list (GO-2026-5013 through GO-2026-5033)
- https://nvd.nist.gov/vuln/detail/CVE-2026-39821 (Punycode CVE details)
- https://go.dev/issue/78760 (idna tracking issue)
- https://go.dev/cl/770080 (NTUnicodeString fix CL)


---

## Updated from Research (2026-06-25, 00:15 UTC)

### Go 1.27 RC1 (released 2026-06-18) — Security-Relevant Changes

Go 1.27 RC1 was tagged 2026-06-18; final release expected August 2026. Several changes have security implications for Gin services and should be planned for now rather than discovered at GA.

#### 1. `crypto/x509.SystemCertPool` now respects `SSL_CERT_FILE` / `SSL_CERT_DIR` on Windows + Darwin (Go 1.27)

**Behavior change:** On Windows and macOS, `crypto/x509.SystemCertPool()` will now load roots from the paths in the `SSL_CERT_FILE` and `SSL_CERT_DIR` environment variables (if set) and use the **native Go verifier** instead of the platform certificate verification APIs.

**Gin security impact:**

- **Container deployments:** many base images and tooling set `SSL_CERT_FILE` to inject a custom CA bundle. In Go 1.27, this will (correctly) override the platform trust store for the Go process. Audit container build files for stray `SSL_CERT_FILE` env-var inheritance from base layers.
- **Cloud environments:** some Kubernetes admission controllers and service meshes set these vars. If a previously-trusted root CA is in the platform store but not in the `SSL_CERT_FILE` path, outbound `crypto/tls` connections from your Gin service will start failing in Go 1.27.
- **Opt-out:** set `GODEBUG=x509sslcertoverrideplatform=0` to keep Go 1.26 behavior.
- **Testing implication:** if you mock the trust store in tests by setting `SSL_CERT_FILE` to a fixture path, that test fixture will now be authoritative. Make sure the fixture includes only the test CA, not the production system store.

#### 2. 5 TLS GODEBUG settings REMOVED permanently in Go 1.27

These temporary escape hatches are GONE in Go 1.27 (no `GODEBUG` opt-in will bring them back):

- `tlsunsafeekm` (added Go 1.22)
- `tlsrsakex` (added Go 1.22)
- `tls3des` (added Go 1.23)
- `tls10server` (added Go 1.22)
- `x509keypairleaf` (added Go 1.23)

**Gin security impact:**

- If your service terminates TLS for legacy clients that need 3DES or TLS 1.0 server-side, **you must fix the client OR pin to Go 1.26** before upgrading. Most likely nobody in your fleet is relying on these — they were off-by-default dangerous settings.
- If you use the deprecated `Config.Rand` for deterministic TLS testing, switch to `testing/cryptotest.SetGlobalRandom` (Go 1.27 deprecates `Config.Rand`; it will be removed in a future release).
- **Audit checklist before upgrading to Go 1.27:**
  ```bash
  # In your repo
  grep -rE 'tlsunsafeekm|tlsrsakex|tls3des|tls10server|x509keypairleaf' . --include='*.go' --include='*.yaml' --include='*.env' --include='Dockerfile*' --include='*.conf'
  # In your container manifests
  kubectl get deploy -o yaml | grep -E 'tlsunsafeekm|tlsrsakex|tls3des|tls10server|x509keypairleaf'
  ```

#### 3. `bzr` version control support REMOVED from `go` command

The `bzr` (Bazaar) VCS is no longer supported by `go get` / `go mod` for module fetches. Almost certainly no Gin project is affected, but worth a quick `go.mod` audit for any obscure internal Bazaar-hosted dependency.

#### 4. `asynctimerchan` GODEBUG setting REMOVED permanently

`time.Timer` and `time.Ticker` channels are **always unbuffered (synchronous)** in Go 1.27. This is more of a concurrency-correctness issue than a security issue, but it can cause subtle goroutine-leak regressions in services that relied on the old buffered-1 behavior for `select` patterns.

#### 5. GODEBUG recognition for removed settings (Go 1.27 `go` command)

The `go` command will now **fail the build** if a removed GODEBUG setting is present in `go.mod` (`godebug` directive) or in `//go:debug` source comments AND set to an old (non-final-default) value. If set to the final default value, the build still succeeds. This is part of the Go 1 compatibility guarantee but means broken GODEBUG references will become build errors rather than silent fallthroughs.

#### 6. `runtime/pprof` goroutine labels in tracebacks (Go 1.27+ modules)

Tracebacks for modules with `go 1.27` directive now include `pprof` goroutine labels in the header line. **Security concern:** goroutine labels can contain sensitive data (request paths with PII, auth tokens, tenant IDs, request bodies). If you set labels via `pprof.SetGoroutineLabels` and have any kind of crash-log forwarding, those labels will now be included in every traceback.

**Opt-out (kept indefinitely for this exact reason):**
```go
// In your main, before the HTTP server starts:
os.Setenv("GODEBUG", "tracebacklabels=0")
```

### Section title updated
- (none — this is a new section appended below the 2026-06-22 update)

### Why this matters for Gin developers
- **Most of these are security-positive removals** — the `tls3des` / `tls10server` / `tlsrsakex` removals tighten the TLS attack surface.
- **The `SSL_CERT_FILE` behavior change on Windows/Darwin is the most likely to bite** in production deployments, especially container/Kubernetes setups with shared base images.
- The RC1 announcement thread on [golang-announce](https://groups.google.com/g/golang-announce/c/Cu9HkstbtpA) is the authoritative source. Run `govulncheck ./...` and `go1.27rc1 test ./...` against your Gin services before the August GA to surface issues early.

### Sources
- https://go.dev/doc/go1.27 (Go 1.27 release notes — official source for all changes above)
- https://raw.githubusercontent.com/golang/website/master/_content/doc/go1.27.md (release notes source, audited 2026-06-25 00:15 UTC)
- https://raw.githubusercontent.com/golang/go/release-branch.go1.27/VERSION (RC1 tag confirmed: `go1.27rc1\ntime 2026-06-18T17:05:58Z`)
- https://github.com/golang/go/issues/78779 (Go 1.27 release notes tracking — open, no `okay-after-rc1` label yet as of 2026-06-25)
- https://groups.google.com/g/golang-announce/c/Cu9HkstbtpA (RC1 announcement — referenced by 2026-06-21 byteiota.com coverage)
- https://byteiota.com/go-1-27-rc1-generic-methods-land-heres-what-changes-now (RC1 community coverage)


## Updated from Research (2026-06-25, 06:09 UTC)

### Gin Context data race — PR #4660 (correctness/security-relevant)

A reproducible data race was documented in PR [#4660](https://github.com/gin-gonic/gin/pull/4660) (open since 2026-05-22, updated 2026-06-25). The race is between `gin.Context.Set()` / `gin.Context.Keys` writes and `gin.Context.Copy()` reads, when `Copy()` is invoked from a goroutine while another goroutine writes keys via `Set()`.

**Reproducer** (from the PR body, simplified):
```go
e.GET("/hello", func(ctx *gin.Context) {
    var wg sync.WaitGroup
    wg.Add(1)
    go func() {
        defer wg.Done()
        for i := 0; i < 20; i++ {
            ctx.Set("key", i)   // map write
        }
    }()
    wg.Add(1)
    go func() {
        defer wg.Done()
        for i := 0; i < 20; i++ {
            _ = ctx.Copy()      // map read
        }
    }()
    wg.Wait()
})
```

`go run -race` reports `WARNING: DATA RACE`. Root cause: `Context.Copy()` performs a shallow copy of the `Keys map[any]any` reference without locking. PR #4695 (merged 2026-06-22) fixed `Errors` and `Accepted` copying but **missed `Keys`** — the underlying map reference is shared, so concurrent write + read races on the map internals.

### Severity

**Medium-high** for Gin services that spawn goroutines from request handlers and use `ctx.Set()` / `ctx.Get()` / `ctx.Copy()` across those goroutines. Concrete impact scenarios:

1. **Map corruption** — concurrent map read + write is a fatal Go runtime panic in Go 1.21+ (`fatal error: concurrent map read and map write`). Under `go run -race` this is detected; under production builds it can crash the process.
2. **Information disclosure via torn reads** — if a write to `ctx.Keys` is partially visible to another goroutine during `Copy()`, the read may observe a half-written `any` interface value, causing a `runtime.Error: invalid memory address or nil pointer dereference` (or worse, leaking bits of other goroutines' memory if the type assertion path is exploitable).
3. **Race-detector false negatives in test** — without `-race`, the race often goes undetected for months until production load triggers it.

### Affected Gin versions

- **All versions up to and including v1.12.0** (current stable, released 2026-02-28) are affected.
- **Gin v1.13** (in development, ~65.7% milestone completion) — **NOT** in the v1.13 milestone. Will not be fixed in v1.13 unless the milestone is re-scoped.
- **No CVE assigned** as of this writing. Maintainers have not yet committed to a fix.

### Audit checklist

For each Gin handler in your service, check:
- [ ] Does the handler spawn goroutines (e.g. `go func() { ... ctx.X(...) ... }()`)?
- [ ] Do those goroutines call `ctx.Set()`, `ctx.Get()`, `ctx.MustGet()`, `ctx.Copy()`, or `ctx.Keys`?
- [ ] Is there any path where one goroutine writes while another reads?

If yes to all three, your service is affected. Fix options (in order of preference):

1. **Per-goroutine context**: don't share `*gin.Context` across goroutines. Pass `context.Context` instead and attach keys to the new context.
2. **Application-level mutex**: wrap `ctx.Keys` access in a `sync.Mutex` in your service code.
3. **Cherry-pick PR #4660** (once it lands upstream) or apply the patch manually — it's small, ~10 lines around `Context.Copy()`.

### Race-detector sweep

Run on every affected service:
```bash
go build -race -o myservice-race ./cmd/myservice
# load-test with realistic concurrent traffic
hey -n 10000 -c 100 http://localhost:8080/endpoint
```

If no `WARNING: DATA RACE` is reported after a representative load test, your service is likely safe in production (modulo the standard caveat that race-detector coverage is not exhaustive).

### Why this matters here

`security.md` is for correctness/availability issues that could affect a Gin service's security posture, not just CVEs. A production crash from a map race is a denial-of-service incident. Information leakage from torn reads is a confidentiality incident. Both fall under "security" in the broader sense, and `ctx.Set` + goroutines is a common pattern in fan-out handlers (e.g. aggregating multiple backend calls).

### Sources
- https://github.com/gin-gonic/gin/pull/4660 (PR body with reproducer, race detector output, codecov report)
- https://github.com/gin-gonic/gin/pull/4695 (predecessor that fixed `Errors`/`Accepted` but missed `Keys`; merged 2026-06-22)
- https://go.dev/wiki/GoMap ("Maps are not safe for concurrent use" — Go runtime fatal error reference)


---

## Updated from Research (2026-06-26, 18:19 UTC)

### Go 1.25.12 / 1.26.5 imminent — two new Security-labeled CLs detected in past 6h

A live re-pull of `https://dev.golang.org/release` at 2026-06-26 18:24 UTC (cycle start) found **two new Security-labeled CLs added to the patch-release dashboards since the 12:14 UTC cycle**. Both are non-embargoed (public-track per Go Security Policy). This update documents the upstream disclosures and adds Gin-impact context for each.

### Deltas from prior cycle (2026-06-26, 12:14 UTC)

| Release dashboard | Prior count (12:14 UTC) | Now (18:24 UTC) | Delta | New CLs |
|---|---|---|---|---|
| Go1.25.12 | 4 | **5** | **+1** | **#80174** `crypto/tls: omit PSK in ECH outer client hello [1.25 backport]` |
| Go1.26.5 | 8 | **10** | **+2** | **#80175** `crypto/tls: omit PSK in ECH outer client hello [1.26 backport]`; **#80154** `html/template: iframe srcdoc attribute not properly escaped` |
| Go1.27 | 271 | 271 | 0 | — |
| Go1.28 | 95 | 95 | 0 | — |
| Pending CLs | 5418 | **5429** | **+11** | mixed (not individually fetched) |
| Pending Proposals | 1229 | **1228** | **−1** | one proposal closed |

### New CVE entry added to "Recent Gin/Go CVEs (May–June 2026)": CVE-2026-42505

Full entry now in security.md (between CVE-2026-39822 and CVE-2026-39821). Summary:

- **CVE-2026-42505** — `crypto/tls` PSK in ECH outer client hello. Public-track (issue [#79282](https://github.com/golang/go/issues/79282)); fix backports opened 2026-06-26 ([#80174](https://github.com/golang/go/issues/80174) for 1.25, [#80175](https://github.com/golang/go/issues/80175) for 1.26). NVD record not yet populated as of 2026-06-26 (CVE reserved by Go Security Team). Severity Low–Medium: on-path attacker can fingerprint SNI+PSK pair from outer ClientHello, but PSK itself is not leaked.
- **Gin exposure**: requires `tls.Config` to have BOTH PSK configured AND ECH negotiation enabled (niche combination — zero-trust mTLS / mesh sidecars only). Default Gin deployments are **NOT** affected.
- **Workaround until Go 1.25.12 / 1.26.5 ship**: set `tls.Config.ECHConfigs = nil` if you have `tls.Config.PSK` configured.
- **Related CLs**: #80174 (1.25 backport), #80175 (1.26 backport). Both CherryPickCandidate + Security labeled.

### html/template iframe srcdoc escape — security hardening, no CVE yet

- **Issue**: [#80154](https://github.com/golang/go/issues/80154) `html/template: iframe srcdoc attribute not properly escaped`. Labels: `Security`, `NeedsFix`. Milestone: Go1.26.5 (not 1.25.12).
- **Impact**: When `iframe srcdoc` action content is placed in a Go html/template, the content is not treated as HTML for escaping purposes. The Go Security Team's own triage note in the issue body says: *"we are treating this as a security hardening issue"* — low exploitability (template author must explicitly place attacker-controlled input into a srcdoc action), but the fix is in the same 1.26.5 release.
- **Gin exposure paths**: (1) any handler using `c.HTML()` with a template that contains `<iframe srcdoc="{{.X}}">` (or equivalent `template.HTML` / `template.JS` interop); (2) admin/CMS UIs that let users embed arbitrary HTML inside iframe sandboxes. For most Gin apps using html/template purely for read-only views, this is **NOT** exploitable because the template doesn't contain a `srcdoc` action at all.
- **No CVE assigned** as of 2026-06-26 18:24 UTC (issue created 2026-06-25 17:54 UTC; Security + NeedsFix labels applied but no CVE reservation by Go Security Team). Will be auto-documented if a CVE is assigned before Go 1.26.5 ships.
- **Gin mitigation pattern**: avoid passing user input into `srcdoc` attributes. If you must, post-escape with `html.EscapeString()` before interpolation:

  ```go
  // UNSAFE — Gin user-supplied HTML in iframe srcdoc
  c.HTML(200, "embed.html", gin.H{"src": userHTML})

  // SAFER — escape before interpolating into srcdoc context
  c.HTML(200, "embed.html", gin.H{"src": html.EscapeString(userHTML)})
  ```

### Gin dashboard deltas (also worth noting)

- **Gin v1.13 milestone: 23/35 → 24/35 (~68.6%)** — ONE new merge since 12:14 UTC cycle: **#4717** `docs: fix BindXML comment referencing nonexistent binding.BindXML` (merged 2026-06-26 16:48:16Z). Trivial 1-line docstring fix (`binding.BindXML` → `binding.XML`); doesn't change behavior but closes a long-standing documentation gap. Remaining open count: **11** (was 12). Milestone still due 2026-06-30 (4 days).
- **Gin master commits in past 6h**: 1 (#4717 docs fix only). No source code commits.
- **Gin PRs touched in past 6h**: 0 (no comments, no new pushes).
- **Gin v1.12.0** still current stable. No v1.13 release tagged.

### Why this matters for Gin developers

1. **The Go 1.25.12 / 1.26.5 patch release window is narrowing.** Two new Security-labeled CLs added in past 6 hours, plus the previously-tracked CVE-2026-39822 fix. Release likely within 24–72 hours.
2. **CVE-2026-42505 affects a narrow but real slice of Gin services**: any handler using outbound `http.Client` with `tls.Config.PSK` + `tls.Config.ECHConfigs`. Audit your outbound TLS configurations and disable ECH if PSK is in use.
3. **#80154 html/template hardening**: most Gin apps don't use iframe srcdoc, but template-driven admin UIs / CMSes should audit for `srcdoc` action usage.
4. **Gin v1.13 milestone at 68.6%, 4 days from due-date** — 11 open PRs remaining. Closing this milestone cleanly will require landing several in-flight PRs (#4712 render/<format> subpackages, #4701 AbortedByHandler, #4696 rune-boundary safety, #4674 url.PathUnescape, etc.) before 2026-06-30.

### Sources for this update

- `https://dev.golang.org/release` (live fetch 2026-06-26 18:24 UTC — `5 Go1.25.12`, `10 Go1.26.5`, `271 Go1.27`, `95 Go1.28`, `5429 Pending CLs`, `1228 Pending Proposals`)
- `https://github.com/golang/go/issues/79282` (CVE-2026-42505 public-track disclosure)
- `https://github.com/golang/go/issues/80154` (html/template iframe srcdoc Security+NeedsFix, Go1.26.5)
- `https://github.com/golang/go/issues/80174` (Go 1.25 backport for CVE-2026-42505, Security+CherryPickCandidate)
- `https://github.com/golang/go/issues/80175` (Go 1.26 backport for CVE-2026-42505, Security+CherryPickCandidate)
- `https://github.com/gin-gonic/gin/milestone/28` (v1.13 — verified 2026-06-26 18:24 UTC: **24/35 closed, ~68.6%**, due 2026-06-30)
- `https://github.com/gin-gonic/gin/pull/4717` (BindXML doc fix; merged 2026-06-26 16:48:16Z)
- `https://github.com/golang/go/issues/80154` milestone: `Go1.26.5` (verified via API)
- `https://cveawg.mitre.org/api/cve/CVE-2026-42505` (verified 2026-06-26 18:24 UTC — `CVE_RECORD_DNE`, not yet populated)
- `https://services.nvd.nist.gov/rest/json/cves/2.0?cveId=CVE-2026-42505` (verified 2026-06-26 18:24 UTC — `totalResults: 0`)
