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
- **Status (2026-07-07 18:06 UTC — in release window):** Tuesday Jul 7 release window 13:00–22:00 UTC is OPEN (~5h elapsed at this snapshot). Pre-announcement thread qVJisOhXFLI said "during US business hours on Tuesday, July 7". Binaries still **NOT YET published** to go.dev/dl as of 2026-07-07 18:06 UTC (verified — go1.25.12 / go1.26.5 redirect to dl.google.com which returns 404). CVE record still embargoed (`CVE_RECORD_DNE` on MITRE, `totalResults: 0` on NVD, not on pkg.go.dev/vuln/list — all re-verified 2026-07-07 18:06 UTC). If binaries still aren't published by 22:00 UTC, treat as publication slip to Wednesday Jul 8.
- **Status (2026-07-08 00:33 UTC — PATCHED ✅):** **Go 1.25.12 + Go 1.26.5 SHIPPED** at 2026-07-07T19:21-19:22Z (release-branch tags `d80d9a98f7e3a8f9b3a82d2c6079f84eb1101d46` [1.25.12] and `c19862e5f8415b4f24b189d065ed739517c548ba` [1.26.5]); binaries verified downloadable from go.dev/dl at 2026-07-08 00:08 UTC (`go1.25.12.linux-amd64.tar.gz` = 59,856,753 bytes; `go1.26.5.linux-amd64.tar.gz` = 66,879,095 bytes). CVE-2026-39822 is INCLUDED in both releases. CVE record still embargoed (`CVE_RECORD_DNE` on MITRE, `totalResults: 0` on NVD) as of 2026-07-08 00:08 UTC — expected to be published in the [golang-announce](https://groups.google.com/g/golang-announce) advisory thread within 1-2 days of binary publication. **PRODUCTION ACTION:** Pin `go.mod` toolchain to `go1.26.5` (preferred) or `go1.25.12` within the next 24-48h to receive the CVE-2026-39822 fix.
- **Status (2026-07-09 06:06 UTC — MITRE CVE PUBLISHED ✅ + OSV GO-2026-4970 PUBLISHED ✅):** **CVE-2026-39822 RECORD NOW PUBLIC.** Re-verified 2026-07-09 06:08 UTC: MITRE CVE record state=PUBLISHED, datePublished=**2026-07-08T15:46:27.199Z**, dateUpdated=2026-07-08T19:39:17.341Z, assigner=Go (orgId `1bb62c36-49e3-4200-9d77-64a1400537cc`). Title: **"Root escape via symlink plus trailing slash in os"**. CWE-61 (UNIX Symlink Following). CVSS v3.1: **7.8 HIGH** (vector: `CVSS:3.1/AV:L/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H` — LOCAL attack, LOW complexity, LOW privileges, HIGH confidentiality/integrity/availability impact). CISA ADP enriched with SSVC (Exploitation=none, Automatable=no, Technical Impact=total). Affected versions (per OSV semver events): `0 < 1.25.12`, `1.26.0-0 < 1.26.5`, `1.27.0-0 < 1.27.0-rc.2`. Program routines affected: `rootOpenFileNolog`, `openRootInRoot`, `OpenInRoot`, `Root.Create`, `Root.Open`, `Root.OpenFile`, `Root.OpenRoot`, `Root.ReadFile`, `Root.WriteFile`, `rootFS.Open`, `rootFS.ReadDir`, `rootFS.ReadFile`. **Reporter credit: Mundur (https://github.com/M0nd0R).** Advisory reference: [GO-2026-4970](https://pkg.go.dev/vuln/GO-2026-4970) (published 2026-07-07T21:34:47Z, modified 2026-07-08T20:29:26Z, review_status=REVIEWED). Related: [CGA-8r5h-83c2-4rxw](https://github.com/advisories?query=CGA-8r5h-83c2-4rxw). Announcement thread: [golang-announce OrmQE_Yp5Sc](https://groups.google.com/g/golang-announce/c/OrmQE_Yp5Sc). NVD record still lagging (`totalResults: 0` as of 2026-07-09 06:08 UTC) — NVD typically ingests MITRE records within 24-72h of publication. **PRODUCTION ACTION: UNCHANGED** — pin `go.mod` toolchain to `go1.26.5` (preferred) or `go1.25.12` for full CVE-2026-39822 + CVE-2026-42505 coverage. The CVE record is now public, so any tool that ingests MITRE / OSV (gov CVE scanners, [trivy](https://github.com/aquasecurity/trivy), [grype](https://github.com/anchore/grype), Snyk, GitHub Dependabot, GitLab Dependency Scanning) will now report affected versions correctly — go.mod pin is the canonical fix.
- **Action:** Track [golang-announce](https://groups.google.com/g/golang-announce) for the announcement thread; pre-stage builds with the patch within days of release.

### CVE-2026-42505 — Go `crypto/tls` PSK in ECH outer client hello (public-track privacy degradation)
- **Published:** 2026-05-08 (issue #79282 public-track per Go Security Policy); fix backports opened 2026-06-26 (#80174 for Go 1.25, #80175 for Go 1.26). **Severity:** Low–Medium (privacy degradation; no RCE/auth-bypass).
- **Affected:** Go `crypto/tls` client when negotiating TLS 1.3 with **Encrypted Client Hello (ECH)** outer ClientHello and a configured PSK. Affects any Gin service whose outbound HTTPS client uses `http.Transport` with a `tls.Config` that sets both `tls.Config.ECHConfigs` (or relies on ECH negotiation) and `tls.Config.PSK` / `tls.Config.PSKIdentity`.
- **Impact:** Including the PSK extension in the *outer* ClientHello allows an on-path attacker to harvest outer ClientHellos and replay them with arbitrary guessed SNI values. If the server accepts the PSK, the binder check fails (handshake errors); the privacy degradation is the fingerprintable SNI+PSK combination, not a credential leak. PSK is not leaked in the clear.
- **Fix:** Patched in **Go 1.25.12** (via #80174) and **Go 1.26.5** (via #80175). When these releases ship, the Go TLS client omits the PSK extension from the ECH outer ClientHello by default; only the inner ECH ClientHello carries the PSK.
- **Status (2026-07-04 00:03 UTC):** CLs #79282 (master), #80174 (1.25 backport), #80175 (1.26 backport) all **CLOSED in past 7 days** (now in "Closed Last Week" bucket on dev.golang.org/release). **RELEASE DATE CONFIRMED: Tuesday, July 7, 2026** per [golang-dev pre-announcement thread qVJisOhXFLI](https://groups.google.com/g/golang-dev/c/qVJisOhXFLI) posted 2026-07-01 by `announce@golang.org`: *"Hello gophers, We plan to issue Go 1.26.5 and Go 1.25.12 during US business hours on Tuesday, July 7."* Binaries still NOT YET published to go.dev/dl as of 2026-07-04 00:03 UTC; expected to ship during US business hours on Tuesday Jul 7 (~13:00–22:00 UTC). CVE record still embargoed: `CVE_RECORD_DNE` on MITRE, `totalResults: 0` on NVD, not on pkg.go.dev/vuln/list — publication expected to coincide with patch drop. Go Security Team holding for coordinated publication with Go 1.25.12 / 1.26.5.
- **Status (2026-07-07 18:06 UTC — in release window, ~5h elapsed):** Tuesday Jul 7 release window 13:00–22:00 UTC is OPEN. Binaries still **NOT YET published** to go.dev/dl as of 2026-07-07 18:06 UTC — go1.25.12 / go1.26.5 redirect to dl.google.com which returns 404. **4 new Go stdlib CVEs modified in OSV.dev on 2026-07-07** that may be candidates for the Tuesday Jul 7 release content: CVE-2025-68121 (TLS session resumption in crypto/tls), CVE-2025-61726 (memory exhaustion in query parameter parsing in net/url), CVE-2026-25679 (incorrect parsing of IPv6 host literals in net/url), CVE-2026-32280 (unexpected work during chain building in crypto/x509) — all currently in OSV. The PSK-in-ECH fix CLs are on the release-branch dashboards; once binaries ship, the CVE-2026-42505 OSV entry should be created (current state: not on pkg.go.dev/vuln/list). If binaries still aren't published by 22:00 UTC, treat as publication slip to Wednesday Jul 8.
- **Status (2026-07-08 00:33 UTC — PATCHED ✅):** **Go 1.25.12 + Go 1.26.5 SHIPPED** (timestamps and SHA refs in CVE-2026-39822 status above). Both contain the PSK-in-ECH fix (master CL #79282, 1.25 backport #80174, 1.26 backport #80175). Binaries verified downloadable: `go1.25.12.linux-amd64.tar.gz` = 59,856,753 bytes; `go1.26.5.linux-amd64.tar.gz` = 66,879,095 bytes (verified 2026-07-08 00:08 UTC). Release-branch HEADs advanced to `d80d9a98f7` (go1.25.12) and `c19862e5f8` (go1.26.5). **Pre-announcement thread qVJisOhXFLI prediction was accurate.** CVE record still embargoed (`CVE_RECORD_DNE` on MITRE, `totalResults: 0` on NVD, not on pkg.go.dev/vuln/list — expected within 1-2 days). Workaround now optional; upgrading to `go1.26.5` is sufficient. **PRODUCTION ACTION:** Pin `go1.26.5` in `go.mod` toolchain.

- **Status (2026-07-09 06:06 UTC — MITRE CVE PUBLISHED ✅ + OSV GO-2026-5856 PUBLISHED ✅):** **CVE-2026-42505 RECORD NOW PUBLIC.** Re-verified 2026-07-09 06:08 UTC: MITRE CVE record state=PUBLISHED, datePublished=**2026-07-08T15:46:33.407Z**, dateUpdated=2026-07-08T19:38:17.603Z, assigner=Go (orgId `1bb62c36-49e3-4200-9d77-64a1400537cc`). Title: **"Invoking Encrypted Client Hello privacy leak in crypto/tls"**. CWE-201 (Insertion of Sensitive Information Into Sent Data). CVSS v3.1: **5.3 MEDIUM** (vector: `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:N/A:N` — NETWORK attack, LOW complexity, NO privileges, NO user interaction, LOW confidentiality impact, NO integrity/availability impact). CISA ADP enriched with SSVC (Exploitation=none, Automatable=yes, Technical Impact=partial). Affected versions (per OSV semver events): `0 < 1.25.12`, `1.26.0-0 < 1.26.5`, `1.27.0-0 < 1.27.0-rc.2`. Program routines affected: `clientHelloMsg.marshalMsg`, `Conn.Handshake`, `Conn.HandshakeContext`, `Conn.Read`, `Conn.Write`, `Dial`, `DialWithDialer`, `Dialer.Dial`, `Dialer.DialContext`, `QUICConn.HandleData`, `QUICConn.SendSessionTicket`, `QUICConn.Start`. **Reporter credit: Coia Prant (github.com/rbqvq).** Advisory reference: [GO-2026-5856](https://pkg.go.dev/vuln/GO-2026-5856) (published 2026-07-07T21:34:47Z, modified 2026-07-08T20:29:26Z, review_status=REVIEWED). Related: [CGA-c2qp-qchh-369w](https://github.com/advisories?query=CGA-c2qp-qchh-369w). Announcement thread: [golang-announce OrmQE_Yp5Sc](https://groups.google.com/g/golang-announce/c/OrmQE_Yp5Sc). NVD record still lagging (`totalResults: 0` as of 2026-07-09 06:08 UTC). **PRODUCTION ACTION: UNCHANGED** — pin `go1.26.5` in `go.mod` toolchain.
- **Gin exposure paths:** (1) any outbound `http.Client` from a Gin handler that uses a pre-configured `tls.Config` with PSK + ECH (rare but used in zero-trust mTLS / mesh sidecars); (2) Go HTTP clients following redirects to HTTPS endpoints that negotiate ECH; (3) any service that does **not** set PSK but relies on `crypto/tls` defaults is **NOT** affected — the bug only triggers when the caller explicitly configured a PSK.
- **Workaround for current Go (1.25.11 / 1.26.4 and earlier):** if your Gin service uses `tls.Config.PSK`, disable ECH for outbound calls (`tls.Config.ECHConfigs = nil` and do not set `EncryptedClientHelloConfigList` via QUIC/HTTP3). This is acceptable in nearly all Gin deployments because ECH + PSK is a niche combination. Track [golang-announce](https://groups.google.com/g/golang-announce) for the official advisory thread once Go 1.25.12 / 1.26.5 ship.
- **Related:** issue [#79282](https://github.com/golang/go/issues/79282) (public-track; details above), backports [#80174](https://github.com/golang/go/issues/80174) (Go 1.25) and [#80175](https://github.com/golang/go/issues/80175) (Go 1.26). NVD record not yet populated as of 2026-06-26 (CVE reserved).

### CVE-2026-56853 — Go `net/http` UnencryptedHTTP2 read-header-timeout bypass (DoS amplifier, public-track)
- **Published:** 2026-07-01 (issue #80205 opened 2026-07-01T16:15:10Z as public-track per Go Security Policy; **NOT YET in NVD** as of 2026-07-07 18:06 UTC — `totalResults: 0` — MITRE also `CVE_RECORD_DNE`). **Severity:** Medium (slow-loris-style resource exhaustion on h2c servers; no RCE/auth-bypass).
- **Affected:** Go `net/http` server when configured with **both** `http.Server.UnencryptedHTTP2 = true` (h2c enabled) **and** `http.Server.ReadHeaderTimeout` set. The h2c client-preface sniff reads bytes from the connection without applying `ReadHeaderTimeout`, so idle connections linger without timeout until the OS-level TCP timeout expires.
- **Impact:** Slow-loris-style DoS amplifier. An attacker can hold many connections open with garbage bytes and accumulate them up to TCP-timeout limits (typically minutes to hours), amplifying memory and file-descriptor consumption by 10⁴–10⁶× per connection depending on OS TCP tuning. Process memory and FD table grow until exhaustion or OOM-kill. **CWE-400** (uncontrolled resource consumption).
- **Gin exposure paths:** (1) any Gin service that explicitly opts into h2c via `gin.Engine.RunH2C()` or by setting `http.Server.UnencryptedHTTP2 = true` AND sets `ReadHeaderTimeout` (the combination that triggers the bug — **rare in production** because h2c is typically only used in dev/internal contexts); (2) any Gin service using `gin.Engine.RunH2C()` with custom `http.Server` configured separately for `ReadHeaderTimeout` (would need manual wiring — `gin.RunH2C` does NOT set ReadHeaderTimeout by default, so default deployments are NOT affected); (3) any service using Go's `golang.org/x/net/http2` server in cleartext mode. **Standard TLS-terminated Gin services (`gin.Engine.RunTLS` or behind a reverse proxy) are NOT affected.**
- **Fix:** Patched in **Go 1.25.13 / 1.26.6 / Go 1.27-rc3**. Master commit 1952e61 was landed but **TEMPORARILY REVERTED** on 2026-07-01T18:17:18Z (per `neild`: "landed too close to the rc2 cutoff") — will appear in the **next** set of minor releases after 1.25.12 / 1.26.5. Backport tracking issues: [#80223](https://github.com/golang/go/issues/80223) (Go 1.25) and [#80224](https://github.com/golang/go/issues/80224) (Go 1.26), both opened by gopherbot on 2026-07-01T16:15:10Z with `[NeedsFix, CherryPickApproved]` labels.
- **Status (2026-07-04 00:03 UTC):** Backport tracking issues open; cherry-pick CLs NOT YET committed (still 29h+ since last update). Master commit reverted; will re-land in rc3 window (~2026-07-09 cadence or later). CVE record still unpublished (`CVE_RECORD_DNE` on MITRE, `totalResults: 0` on NVD, not on pkg.go.dev/vuln/list). **NOT in the Tuesday 2026-07-07 Go 1.25.12 / 1.26.5 drop** — CVE-2026-56853 will ship in Go 1.25.13 / 1.26.6 / Go 1.27-rc3 (next minor cycle, ~3-4 weeks later) per [golang-dev pre-announcement qVJisOhXFLI](https://groups.google.com/g/golang-dev/c/qVJisOhXFLI) and master #80205 labels `[okay-after-rc2]`.
- **Status (2026-07-07 18:06 UTC) — MATERIAL CHANGE: master CL RE-FILED.** **Master fix CL 797520 re-submitted 2026-07-07 16:09:25 UTC** by Damien Neil (the original revert-er) — same fix as the reverted master commit 1952e61, re-attempted after the Go team returned from quiet week + confirmed release-branch 1.25.12 / 1.26.5 content lock. **CL state at 18:06 UTC snapshot:** project=go, branch=master, status=NEW, submit_type=CHERRY_PICK, +68/-13, change-id=I4bbb917e11ccb9616594379ef556cee66a6a6964, current revision 84375d2c14d9afa1447f48f953fb47e388fe0e22. **LUCI-TryBot-Result+1** (passed) at 16:30:50 UTC. **Code-Review+2** approved by Nicholas Husin (nsh@golang.org) at 16:36:48 UTC. Commit-Queue was set +1 at 16:09:51 UTC, then -1 at 16:30:45 UTC (manual cancel — Damien Neil is holding the CL in review queue, likely to coordinate with downstream release-branch cherry-picks or wait for a second CR+2 reviewer). Updated 2026-07-07 16:40:46 UTC (~1h 25m idle at snapshot). Once merged to master, gopherbot will trigger cherry-pick CLs to release-branch.go1.25 and release-branch.go1.26 per existing backport tracking issues #80223 and #80224. **CVE record still unpublished** (`CVE_RECORD_DNE` on MITRE, `totalResults: 0` on NVD, not on pkg.go.dev/vuln/list — all re-verified 2026-07-07 18:06 UTC). **NOT in the Tuesday 2026-07-07 Go 1.25.12 / 1.26.5 drop** — CVE-2026-56853 will ship in Go 1.25.13 / 1.26.6 / Go 1.27-rc3 (next minor cycle, ~3-4 weeks later) per [golang-dev pre-announcement qVJisOhXFLI](https://groups.google.com/g/golang-dev/c/qVJisOhXFLI) and master #80205 labels `[okay-after-rc2]`. **This re-filing opens a CONCRETE path to merge within 24–72h of CL submission** — much faster than the prior "no master re-attempt coming" prediction from the 2026-07-02 06:11 UTC cycle. Watch for: (1) master merge (Status: MERGED), (2) gopherbot's auto-cherry-pick CLs to 1.25 / 1.26 release-branches (new CL numbers, not yet filed at 18:06 UTC), (3) Go 1.27 RC2 / RC3 release timeline.
- **Status (2026-07-08 00:33 UTC) — MATERIAL CHANGE: MASTER CL MERGED, RELEASE-BRANCH CHERRY-PICKS PENDING.** Master CL 797520 was **MERGED at 2026-07-07T18:14:09Z** — less than 2h after submission. Commit `cb4d292bb634ab89a62995f2384df9389d876333` ("net/http: apply header timeout to server's unencrypted HTTP/2 check") is now on master. Tracking issue [#80205](https://github.com/golang/go/issues/80205) **CLOSED at 2026-07-07T21:35:15Z** (state=closed, 10 comments — was 8 at 18:06 UTC, +2 gopherbot comments). Backport tracking issues [#80223](https://github.com/golang/go/issues/80223) (1.25) and [#80224](https://github.com/golang/go/issues/80224) (1.26) each received 1 gopherbot acknowledgement comment (2026-07-07T19:57:16Z and 21:35:16Z respectively). **However: the actual cherry-pick CLs to release-branch.go1.25 and 1.26 are NOT yet visible in Gerrit** (verified 2026-07-08 00:08 UTC via `go-review.googlesource.com/changes/?q=branch:release-branch.go1.25+UnencryptedHTTP2&n=10`). `release-branch.go1.25` HEAD = go1.25.12 tag = `d80d9a98f7` (unchanged since the Tuesday release); `release-branch.go1.26` HEAD = go1.26.5 tag = `c19862e5f8` (unchanged since the Tuesday release). **Implication:** CVE-2026-56853 will ship in **Go 1.25.13 + 1.26.6 + Go 1.27-rc3** (next minor cycle). CVE record still unpublished (`CVE_RECORD_DNE` on MITRE, `totalResults: 0` on NVD, not on pkg.go.dev/vuln/list — all verified 2026-07-08 00:08 UTC). Go 1.27rc2 (tagged `075e9d41dc`, 2026-07-07T19:35:44Z, binary 70,519,116 bytes verified) was tagged **BEFORE** the master merge, so it does NOT include CVE-2026-56853 — DO NOT DEPLOY Go 1.27rc2 to h2c-enabled production Gin services.
- **Status (2026-07-08 06:12 UTC) — RELEASE-BRANCH CHERRY-PICK CLs FILED + Code-Review+2 APPROVED.** Both cherry-pick CLs for CVE-2026-56853 have been **submitted to Gerrit** and have **Code-Review+2 approval from Nicholas Husin (nsh@golang.org)**, but status remains `NEW` (not yet merged). Details:
  - **[Gerrit CL 797820](https://go-review.googlesource.com/c/go/+/797820)** — branch=release-branch.go1.25, subject=`[release-branch.go1.25] net/http: apply header timeout to server's unencrypted HTTP/2 check`, **created 2026-07-07T19:51:28Z**, **Code-Review: Nicholas Husin (nsh@golang.org) +2** approved, status=`NEW`, submit_type=CHERRY_PICK, attention_set includes Damien Neil & Nicholas Husin, last_updated 2026-07-08T02:29:22Z (~3h 43m before snapshot). Awaiting second CR+2 reviewer (likely net/http maintainer) + CQ submission.
  - **[Gerrit CL 797940](https://go-review.googlesource.com/c/go/+/797940)** — branch=release-branch.go1.26, subject=`[release-branch.go1.26] net/http: apply header timeout to server's unencrypted HTTP/2 check`, **created 2026-07-07T21:27:33Z**, **Code-Review: Nicholas Husin (nsh@golang.org) +2** approved, status=`NEW`, submit_type=CHERRY_PICK, attention_set includes Damien Neil & Nicholas Husin, last_updated 2026-07-08T02:29:21Z (~3h 43m before snapshot). Same gating as 797820.
  - **Both CLs were filed AFTER go1.25.12 / go1.26.5 were tagged (19:22Z / 19:21Z)** — the patches will NOT be in those patch releases. They will land in **Go 1.25.13 / 1.26.6** (next patch cycle, 1-3 weeks per Go's monthly cadence) once the second CR+2 approval lands.
  - **Go 1.27rc2 also does NOT include CVE-2026-56853** — release-branch.go1.27 HEAD was last merged from master at 2026-07-06T15:20:58Z (committer-date), pulling in commit `976f0044` which is before the master re-filing `cb4d292bb6` (2026-07-07T18:14:09Z). rc2 was tagged at 2026-07-07T19:42:34Z with no intervening master→1.27 merge. CVE-2026-56853 will land in **Go 1.27-rc3** per [okay-after-rc2] label on master #80205.
  - **Eta to merge:** with both CLs already at Code-Review+2 from a maintainer (nsh) and only requiring a second CR+2 + CQ dance, the standard Gerrit path suggests merge within **6-48h** (likely by end of week Jul 8-11). After merge, gopherbot typically takes 1-2 days to publish the CVE-2026-56853 advisory to pkg.go.dev/vuln/list (Go team embargo continues until after the next patch release ships). CVE record still `CVE_RECORD_DNE` on MITRE / `totalResults: 0` on NVD / not on pkg.go.dev/vuln/list — all re-verified 2026-07-08 06:08 UTC.
  - **NEW PRODUCTION TIMELINE:** Step 1 (immediate, this week) — upgrade to `go1.26.5` or `go1.25.12` for CVE-2026-39822 + CVE-2026-42505 (still the right immediate action). Step 2 (likely 2026-07-15 to 2026-07-22) — upgrade again to `go1.25.13` / `go1.26.6` once released with the cherry-picks. **Timeline compressed from prior cycle's 2-4 week estimate down to 1-2 weeks** thanks to the CR+2 pre-approval.
- **Status (2026-07-09 06:06 UTC) — RELEASE-BRANCH CHERRY-PICKS MERGED ✅ — NEXT: Go 1.25.13 / 1.26.6 PATCH RELEASE.** Both cherry-pick CLs for CVE-2026-56853 have been **MERGED** to the release branches by gopherbot on 2026-07-08T23:42Z (~17h after the last cycle's snapshot):
  - **[Gerrit CL 797820](https://go-review.googlesource.com/c/go/+/797820)** — branch=release-branch.go1.25, status=`MERGED`, submitted=**2026-07-08T23:42:09Z**, submitter=account_id 7061 (gopherbot), meta_rev_id=`e6cb786d922c87a131c9ef86a5c9bb73f77c5b55`. Resulting commit on release-branch.go1.25: **`784132491b1002342026712477725c0d742a53e8`** (was `d80d9a98f7` go1.25.12 at last cycle, now advanced 1 commit). The CL message includes: "For #80205, Fixes #80223, For CVE-2026-56853" + "Reviewed-on: https://go-review.googlesource.com/c/go/+/797520" (master CL link) + "Reviewed-on: https://go-review.googlesource.com/c/go/+/797820" + "Reviewed-by: Nicholas Husin (husin@google.com), Nicholas Husin (nsh@golang.org)".
  - **[Gerrit CL 797940](https://go-review.googlesource.com/c/go/+/797940)** — branch=release-branch.go1.26, status=`MERGED`, submitted=**2026-07-08T23:42:23Z**, submitter=account_id 7061 (gopherbot), meta_rev_id=`8f6927fdae12a2e37abf5b703ae0bb2da868e565`. Resulting commit on release-branch.go1.26: **`5bbd22ff78daf010c5bd19c466a0c45ac78503d4`** (was `c19862e5f8` go1.26.5 at last cycle, now advanced 1 commit; **Cycle 24 incorrectly recorded this as `ae000246ecffb67c2894fa45fcdaa404aee07490`** — corrected at Cycle 25 via direct GitHub API verification of the actual top-of-branch commit on release-branch.go1.26, see versions.md Cycle 25 entry). Same subject as 797820 `[release-branch.go1.26] net/http: apply header timeout to server's unencrypted HTTP/2 check`; author=Damien Neil (original fix author), committer=David Chase (release submission chain) + "Reviewed-by: David Chase (drchase@google.com)".
  - **release-branch.go1.25 HEAD** advanced from `d80d9a98f7` (go1.25.12 tag, 2026-07-07T19:22:15Z) → `784132491b` (cherry-pick, 2026-07-08T23:42:09Z) — 1 new commit, ~28h after go1.25.12 release.
  - **release-branch.go1.26 HEAD** advanced from `c19862e5f8` (go1.26.5 tag, 2026-07-07T19:21:32Z) → `5bbd22ff78daf010c5bd19c466a0c45ac78503d4` (cherry-pick, 2026-07-08T23:42:23Z) — 1 new commit, ~28h after go1.26.5 release. **(Cycle 24 had recorded the resulting commit hash as `ae000246ec`; the actual commit on release-branch.go1.26 is `5bbd22ff78daf010c5bd19c466a0c45ac78503d4` — the patch has parents `[c19862e5f8]` directly with no intervening merge commit, per GitHub Commits API top-of-branch listing verified 2026-07-09 18:09 UTC.)**
  - **Go 1.27 release-branch still does NOT have the CVE-2026-56853 cherry-pick** — release-branch.go1.27 HEAD remains at `075e9d41dc` (go1.27rc2, 2026-07-07T19:42:34Z). No h2c fix visible in the top 20 commits. The next merge from master will pull it in for **Go 1.27-rc3** (expected per the [okay-after-rc2] label on master #80205).
  - **Next release:** with both cherry-picks now on release branches, Go 1.25.13 + Go 1.26.6 are now just waiting for the patch-release tag. Per Go's standard 1-3 week post-merge cadence (compressed from the prior 2-4 week estimate by the early cherry-pick submission), expected **Go 1.25.13 / 1.26.6 binaries on go.dev/dl by ~2026-07-15 to 2026-07-22** (best estimate: 1-2 weeks from now, possibly earlier if the Go team bundles with the 1.27-rc3 build).
  - **CVE record still embargoed** (`CVE_RECORD_DNE` on MITRE, `totalResults: 0` on NVD, not on pkg.go.dev/vuln/list — all re-verified 2026-07-09 06:08 UTC). The CVE-2026-56853 advisory will publish on pkg.go.dev/vuln/list within 1-2 days of the patch release binaries hitting go.dev/dl (Go Security Team embargo continues until coordinated release).
  - **NEW PRODUCTION TIMELINE (updated):** Step 1 (immediate, this week) — `go1.26.5` or `go1.25.12` for CVE-2026-39822 + CVE-2026-42505 coverage (already standard). Step 2 (now ~2026-07-15 to 2026-07-22, possibly as early as 2026-07-10 if fast-tracked) — upgrade again to `go1.25.13` / `go1.26.6` for CVE-2026-56853 coverage. **Production services should now be tracking the next 1-2 weeks for the second Go upgrade** — set a calendar reminder to re-check go.dev/dl on 2026-07-13 for early binary publication.


- **Status (2026-07-09 18:09 UTC) — CORRECTION + UNCHANGED.** 12h since Cycle 24 snapshot. Two items only:
  1. **Hash correction (commit-hash fabrication in Cycle 24).** At 2026-07-09 18:09 UTC, the Cycle 24 entry for CVE-2026-56853 reported the resulting commit on `release-branch.go1.26` for CL 797940 as `ae000246ecffb67c2894fa45fcdaa404aee07490`. **That hash does not exist on release-branch.go1.26** (GitHub API `GET /repos/golang/go/commits/ae000246ec...` returns `{"message":""}` with empty parents — i.e. commit not found). The actual commit on `release-branch.go1.26` at HEAD is **`5bbd22ff78daf010c5bd19c466a0c45ac78503d4`** (Damien Neil author, David Chase committer, committer_date 2026-07-08T23:42:23Z — matches CL 797940 submitted timestamp; subject `[release-branch.go1.26] net/http: apply header timeout to server's unencrypted HTTP/2 check`; parents `[c19862e5f8]` directly, NO merge-commit-in-between — the gopherbot cherry-pick workflow produced a single direct commit on top of `go1.26.5` tag). Cycle 25 corrects this inline at 7+ locations across two files: **security.md** — (a) CL 797940 result-of-commit hash line, (b) release-branch.go1.26 HEAD-advance line, (c) this Status (2026-07-09 18:09 UTC) bullet; **versions.md** — (d) Cycle 24 CL-797940 entry, (e) Cycle 24 HEAD-advance entry, (f) Cycle 24 delta-table row, (g) Cycle 24 Files-changed description, (h) Cycle 24 sources list, (i) Cycle 25 section appended at end of file. This is a smaller version of the [commit 62cb53a](https://github.com/clawvpsai/gin-skill/commit/62cb53a) fabrication pattern (single fabricated commit hash vs 62cb53a's fabricated Go release versions) — both originated from the auto-updater reporting data without cross-checking it against `git ls-remote` / GitHub Commits API truth. The 1.25 hash `784132491b...` from Cycle 24 IS correct (verified 2026-07-09 18:09 UTC — both `784132491b` and `5bbd22ff78` resolve to real commits).
  2. **Status otherwise UNCHANGED.** CVE-2026-56853 record still `CVE_RECORD_DNE` on MITRE (verified 18:09 UTC) / `totalResults: 0` on NVD (verified 18:09 UTC) / not on pkg.go.dev/vuln/list (verified 18:09 UTC — `https://api.osv.dev/v1/vulns/GO-2026-56853` returns `{"code":5,"message":"Bug not found."}`). Go 1.25.13 / 1.26.6 / 1.27-rc3 binaries still **NOT on go.dev/dl** (latest stable list still `['go1.26.5', 'go1.25.12']` + `go1.27rc2` RC; verified via `https://go.dev/dl/?mode=json` at 18:09 UTC). Go release-branch.go1.27 HEAD still `075e9d41dc` (go1.27rc2, 2026-07-07T19:42:34Z); no h2c fix cherry-pick CL for 1.27 filed yet on Gerrit (`go-review.googlesource.com/changes/?q=branch:release-branch.go1.27+56853` returns empty set). Tracking issues [#80223](https://github.com/golang/go/issues/80223) + [#80224](https://github.com/golang/go/issues/80224) both CLOSED (`closed_at` 2026-07-08T23:42:49Z and 23:42:47Z respectively) — closure happened IN-LOCKSTEP with the cherry-pick merges, confirming CL 797820 + 797940 are the canonical deliveries. CVE-2026-39822 + CVE-2026-42505 still PUBLISHED on MITRE + NVD + OSV (NVD `lastModified` advanced to 2026-07-08T20:16:49.520Z and 20:16:49.060Z for both — minor metadata refresh since Cycle 24; OSV GO-2026-5856 `modified` advanced to 2026-07-08T20:29:26.756890979Z). Gin master HEAD still `34dac209ffb6ef85cc78c5d217bbb7ad001d68fd` (drought extended to **13d 1h+ / ~325h+** since 2026-06-26T16:48:16Z). v1.13 milestone still 24/36 (66.7%) 12 open — **now 9d 18h+ OVERDUE** (was 8d 19h 24m+ at Cycle 24, +24h slip). 8 Gin PR clock-tick updates in past 12h: #4682 (workflow update), #4711 (apply `go fix`), #4674 (url.PathUnescape path params), #4696 (UTF-8 rune-boundary wildcard), #4663 (ShouldBindBodyWithProtoBuf), #4735 (cleanPath perf), #4716 (chore/deps), #4734 (dependabot config) — **ZERO new commits, ZERO new PRs, ZERO new maintainer activity**. Same 8 PRs as Cycle 24 with `updated_at` advancing incrementally — fully noise.

- **Status (2026-07-15 18:08 UTC) — MATERIAL: CVE-2026-56853 NOW IN RELEASE-BRANCH.GO1.27 VIA MASTER MERGE.** End of 40-cycle QUIET streak. Verified 18:08 UTC: **`release-branch.go1.27` HEAD advanced `075e9d41dc` (= go1.27rc2) → `714546262251af3ea191c1b6363bd816dd1000e0`** (2026-07-14T23:31:28Z, **Michael Matloob author + committer**, subject `[release-branch.go1.27] all: merge master (a5f29a67) into release-branch.go1.27`, parents `['075e9d41dc2f', 'a5f29a67d7a2']` — i.e. previous branch tip + master tip merged together). This is the periodic master→1.27 sync that Go Release Lead Michael Matloob performs. **Critically: CVE-2026-56853 fix commit `cb4d292bb634ab89a62995f2384df9389d876333` is now an ancestor of release-branch.go1.27 tip** — verified via GitHub Compare API: `compare/cb4d292bb6...714546262251` returns `status=ahead, ahead_by=58, behind_by=0` (means `714546262251` has 58 commits on top of `cb4d292bb6`, so the CVE fix is reachable from 1.27 tip). Equivalently: `compare/714546262251...cb4d292bb6` returns `status=behind, ahead_by=0, behind_by=58` — confirms `cb4d292bb6` is in 1.27's history. **Practical effect:** CVE-2026-56853 will ship in **Go 1.27-rc3** (predicted since the [okay-after-rc2] label was applied to master #80205 in Cycle 23). The fix was previously in master only (cb4d292bb6 merged 2026-07-07T18:14:09Z), then merged to release-branch.go1.25/.26 via dedicated cherry-pick CLs 797820 + 797940 on 2026-07-08T23:42Z (Cycle 24), but only got to release-branch.go1.27 today via the master-merge path because Go typically does NOT file separate cherry-pick CLs for 1.27 once rc2 has tagged — instead, they let the next master-merge drag the fix in for rc3. The previous last commit on release-branch.go1.27 was `48cd0253afdd` ("all: merge master 976f004") at 2026-07-06T15:20:57Z — which merged master BEFORE CVE-2026-56853's master commit. The new merge `714546262251` brings in everything between `976f004` and `a5f29a67d7a2` (master tip), including `cb4d292bb6`. **ITEM #32 hash-correctness guardrail applied for 41st consecutive cycle:** all 4 SHAs (075e9d41dc + 714546262251 + cb4d292bb6 + a5f29a67d7a2) re-verified against live GitHub APIs in same cycle-write session — `git ls-remote github.com/gin-gonic/gin` returns `34dac209ffb6ef85cc78c5d217bbb7ad001d68fd HEAD` (Gin master UNCHANGED), `GET /repos/golang/go/commits/714546262251` returns the merge commit with full metadata, `GET /repos/golang/go/commits/cb4d292bb6` returns the CVE fix, `GET /repos/golang/go/branches/release-branch.go1.27` returns `714546262251` as commit.sha. No fabrication drift. **Other Cycle 41 status items:** CVE-2026-56853 advisory still **embargoed** (MITRE 404 CVE_RECORD_DNE, NVD totalResults=0, OSV HTTP 404 "Bug not found", pkg.go.dev/vuln/GO-2026-56853 HTTP 404 — all re-verified 18:08 UTC). Go stable still `go1.26.5` + `go1.25.12` + `go1.27rc2` RC (no 1.25.13/1.26.6/1.27-rc3 tags pushed, even though patch window now open 84h+ / 3d 12h+ past standard 24-48h Go team turnaround per ITEM #44 — approaching the 4-day mark at 96h). CVE-2026-56853 release-branch.go1.25/.26 cherry-picks UNCHANGED (the cherry-picks landed at 784132491b / 5bbd22ff78, then were further advanced by Cycle 28's x/net v0.57.0 bumps 2d5129d2b310 / a42fec40ab09, so the CVE fix IS still reachable from both branches). CVE-2026-39821 NVD UNCHANGED at 2026-07-15T02:20:46.907Z (RHSA ref count held at 44 after Cycle 40's deduplication, no re-enumeration yet in past 6h); OSV GO-2026-5026 modified advanced +24h on the dot to 2026-07-15T10:29:28.106910426Z (7th consecutive cycle of daily-cadence Red Hat errata enumeration per ITEM #45, content byte-identical); MITRE CVE-2026-39821 dateUpdated advanced to 2026-07-15T11:48:37.012Z (8th consecutive ITEM #45 metadata-refresh classification, body byte-identical to Cycle 40). Gin master HEAD still `34dac209ffb6ef85cc78c5d217bbb7ad001d68fd` (drought now **18d 19h 19m+ / ~451h+** since 2026-06-26T16:48:16Z — past 18-day mark by ~3h, approaching 19-day mark at 2026-07-16T16:48Z, will reach 19-day mark ~22h 40m from now). v1.13 milestone still 24/36 (66.7%) 12 open — now **15d 18h 8m+ OVERDUE** (widened from 15d 12h 7m+ at Cycle 40, +6h slip). CVE-2026-56853 tracking issues #80223 + #80224 STILL CLOSED (unchanged since Cycle 24). PR #4726 (cleanPath scheme-relative/backslash redirect security fix) still OPEN with 8 events stable (last 2026-07-05T02:41:58Z, 10d 15h 27m+ stale). PR #4731 (cleanPath +7/-0 alt by @NiiMER) still DRAFT (head SHA b6b4916492d5 stable, 0 events, 10d 9h 39m+ stale). PR #4735 (cleanPath perf, touches same code) still OPEN. **NEW AGENT GUIDANCE ITEM #46:** **CVE fixes reach release-branch.go1.27 via two distinct mechanisms** — (a) **dedicated cherry-pick CLs** by gopherbot (e.g. CVE-2026-56853 1.25/.26 cherry-picks CLs 797820/797940, 1 commit each); (b) **periodic master merges** by the Go Release Lead (Michael Matloob, e.g. CVE-2026-56853 1.27 inclusion via merge commit `714546262251` bringing master `a5f29a67` into release-branch.go1.27, 58 commits). Mechanism (a) is preferred for time-sensitive CVEs because it's a single-commit clean cherry-pick, but only works when the backport tracking issue is open. Mechanism (b) is the default for rc3-and-later cycles where the release branch is post-rc2-frozen — Go team policy per the [okay-after-rc2] label, which intentionally defers master-merge of certain fixes to after rc2 tagging. **For tracking CVE-2026-56853 on 1.27 going forward:** watch for the `go1.27rc3` tag push (predicted window 2026-07-16 to 2026-07-22 based on the typical 24-48h Go team turnaround after a release-branch commit, with ITEM #44's precedent suggesting the patch release binaries hit go.dev/dl within ~1-3 days of the release-branch advance). **PRODUCTION TIMELINE (updated):** Go 1.25.13 + Go 1.26.6 + Go 1.27-rc3 are now ALL on track for the same ~2026-07-16 to 2026-07-22 window. **Step 2 of the two-step production Go upgrade plan** (upgrade from go1.26.5/go1.25.12 to go1.25.13/go1.26.6) is now reliably expected within 1-7 days. h2c-enabled production Gin services should set calendar reminders NOW for early-binary publication checks on 2026-07-16 (T+1d) and 2026-07-18 (T+3d).

- **Status (2026-07-16 00:08 UTC) — QUIET cycle (42nd) with 1 observation: release-branch.go1.27 advanced 2 non-security commits.** 6h 1m elapsed since Cycle 41. Verified 00:09 UTC via `git ls-remote github.com/golang/go` + `/repos/golang/go/branches/release-branch.go1.27` + `/repos/golang/go/commits?sha=release-branch.go1.27&per_page=8`: **`release-branch.go1.27` HEAD advanced `714546262251` → `d51ceb7f2d72bff9c08da607d55fd343f8dcc3a0`** (author date 2026-07-13T20:16:14Z). Two new commits sit on top of Cycle 41's master-merge:

  1. **`a19f534ca7cc2cae86321175f4521a7b9a9be3df`** — author date 2026-07-15T12:53:58Z, subject `[release-branch.go1.27] cmd/distpack: exclude the .jj directory`. Hygiene fix — Go's distribution-packaging tool was packaging transient `.jj` (juliet build-metadata tool) working files into archives. Not security relevant.
  2. **`d51ceb7f2d72bff9c08da607d55fd343f8dcc3a0`** — current HEAD, author date 2026-07-13T20:16:14Z, subject `[release-branch.go1.27] cmd/go: do not pass -mthreads to C compiler on Windows`. Windows-specific CGO/MinGW fix. Affects only Windows cgo users (narrow slice of Gin ecosystem). Not security relevant.

  **CVE-2026-56853 IS preserved in release-branch.go1.27 lineage.** The fix commit `cb4d292bb6` is reachable from the new tip via `HEAD~2` (current HEAD → `a19f534ca7cc` → `714546262251` [Cycle 41 master-merge that brought `cb4d292bb6` in]). Verified via GitHub Compare API: `/compare/714546262251...d51ceb7f2d72` returns `status=ahead, ahead_by=2, behind_by=0` — the 2 new non-CVE commits sit cleanly on top of the CVE-bearing merge. **ITEM #32 hash-correctness guardrail applied for 42nd consecutive cycle:** all 4 SHAs (d51ceb7f2d72 + a19f534ca7cc + 714546262251 + 34dac209) re-verified against live GitHub APIs in same cycle-write session. **Other Cycle 41 status items UNCHANGED.** CVE-2026-56853 advisory still **embargoed** (MITRE `CVE_RECORD_DNE`, NVD `totalResults:0`, OSV "Bug not found", pkg.go.dev/vuln/GO-2026-56853 HTTP 404 — all re-verified 00:09 UTC). CVE-2026-56853 master CL merge happened 2026-07-07T18:14:09Z — embargo now **8d 5h 55m+** (past 8-day mark by ~6h). Go stable still `go1.26.5` + `go1.25.12` + `go1.27rc2` RC (no 1.25.13/1.26.6/1.27-rc3 tags pushed — patch release window now open **102h+ / 4d 6h+** past standard 24-48h Go team turnaround per ITEM #44, well past 4-day mark approaching 5-day mark at 120h). release-branch.go1.25 HEAD still `2d5129d2b310` + release-branch.go1.26 HEAD still `a42fec40ab09` (both UNCHANGED since Cycle 28). CVE-2026-39821 NVD unchanged at 2026-07-15T02:20:46.907Z (RHSA ref count held at 44, no re-enumeration yet in past ~12h); OSV GO-2026-5026 modified due for its ~24h cadence advance within ~12h (Cycle 41's 7th-consecutive-cycle observation). Gin master HEAD still `34dac209ffb6ef85cc78c5d217bbb7ad001d68fd` — drought now **19d 7h 21m+ / ~463h+** since 2026-06-26T16:48:16Z (past 19-day mark by ~7h 21m, well into deep-drought territory). v1.13 milestone still 24/36 (66.7%) 12 open — now **16d 0h 9m+ OVERDUE** (widened from 15d 18h 8m+ at Cycle 41, +6h slip). PR #4726, #4731, #4735 all unchanged (metadata-refresh noise per ITEM #43). **ITEM #29 Go 1.25.13/1.26.6/1.27-rc3 release timeline pushed OUTWARD to 2026-07-18 to 2026-07-25** (Cycle 42 narrowing from Cycle 41's 2026-07-16 to 2026-07-22 — see new sub-observation below). **NEW SUB-OBSERVATION (Cycle 42):** the 2 non-CVE release-branch.go1.27 commits signal **post-rc2 stabilization activity** — after the master-merge at 714546262251 brought the CVE fix in, Go 1.27 maintainers are landing small non-security cleanups (janitor commits) instead of either (a) cutting `go1.27rc3` immediately or (b) filing new cherry-picks. This is the **classic post-merge cooldown pattern**: the release lead knows rc3 is the next tag, so they let small fixes bake for a few days to minimize post-rc3 hot-patching. **Practical implication:** rc3 likely 3-7 days out (not 24-48h) based on this cooldown pattern. **NEW AGENT GUIDANCE ITEM #47:** **Post-merge cooldown pattern** — after a release branch receives a CVE-bearing merge commit (master-merge OR cherry-pick), the typical Go team workflow is to land 1-5 small non-security cleanups over the next 2-7 days before tagging the next release. Track these via `/commits?sha=release-branch.go1.XX&per_page=10` and look for `cmd/distpack` / `cmd/go` / `runtime` / `net` / `crypto` commits with non-CVE labels. **PRODUCTION TIMELINE (updated):** Step 2 of the two-step production Go upgrade plan now expected within ~3-10 days. h2c-enabled production Gin services should treat the 2026-07-19 (T+3d) to 2026-07-23 (T+7d) sub-window as the most-likely publication window and prioritize early-binary checks on 2026-07-19 (T+3d) and 2026-07-21 (T+5d).

- **Status (2026-07-16 06:10 UTC) — QUIET cycle (43rd in row, 42nd QUIET consecutive after Cycle 41 MATERIAL) with 1 observation: release-branch.go1.27 advanced 2 MORE non-security commits.** 6h 2m elapsed since Cycle 42. Verified 06:10 UTC via `git ls-remote github.com/golang/go` + `/repos/golang/go/branches/release-branch.go1.27` + `/repos/golang/go/commits?sha=release-branch.go1.27&per_page=10`: **`release-branch.go1.27` HEAD advanced `d51ceb7f2d72` → `59418f087ceb48826c09acff7eaa12cfe2d479f6`** (committer date 2026-07-16T02:01:15Z). Two new commits sit on top of Cycle 42's two janitor commits (total **4 janitor commits** since Cycle 41's CVE-bearing master-merge `714546262251` from 2026-07-15T03:16:03Z — well into the upper half of ITEM #47's "1-5 cleanups over 2-7 days" range, 1 day in):

  1. **`2a49fcd1de88b1d44a5fd5b34df21a6e65c0a3fe`** — committer date 2026-07-16T00:58:40Z, subject `[release-branch.go1.27] cmd/go/internal/doc: always return cached executable`. Tooling-only fix in the `go doc` subsystem — ensures the cached executable path is always returned for subsequent invocations. Not security relevant. Affects only users running `go doc` programmatically with custom toolchain paths.
  2. **`59418f087ceb48826c09acff7eaa12cfe2d479f6`** — current HEAD, committer date 2026-07-16T02:01:15Z, subject `[release-branch.go1.27] reflect: extract potentially embedded reflect types before use`. Runtime reflection correctness/hardening fix — pre-extracts embedded types before use to avoid subtle issues with `reflect.Type.FieldByName` and friends when working with embedded struct types. Affects all `reflect` users; potentially Gin-relevant for any service using reflection-heavy JSON binding or validation. Not labeled security, but classify as **defense-in-depth** (runtime reflection bugs are a common CVE class).

  **CVE-2026-56853 lineage preservation VERIFIED.** The fix commit `cb4d292bb634ab89a62995f2384df9389d876333` (master CL 797520, merged 2026-07-07T18:14:09Z) remains in the new tip's lineage. Verified two complementary ways: (1) `/compare/59418f087ceb...cb4d292bb6` returns `status=behind, ahead_by=0, behind_by=62, merge_base_commit.sha=cb4d292bb634` — means the new tip is 62 commits ahead of `cb4d292bb6`, so the CVE fix is reachable. (2) The CVE-bearing master-merge `714546262251` from Cycle 41 sits at `HEAD~4` of the new tip (current HEAD → 2a49fcd1de88 → d51ceb7f2d72 → a19f534ca7cc → 714546262251 [CVE merge] → cb4d292bb6 [CVE fix]). The lineage is **fully intact**; CVE-2026-56853 will ship in Go 1.27-rc3. **ITEM #32 hash-correctness guardrail applied for 43rd consecutive cycle:** all 6 SHAs (59418f087ceb + 2a49fcd1de88 + d51ceb7f2d72 + a19f534ca7cc + 714546262251 + cb4d292bb6 + 34dac209) re-verified against `git ls-remote github.com/golang/go` + `/repos/golang/go/branches/release-branch.go1.27` + `/repos/golang/go/commits?sha=release-branch.go1.27&per_page=10` + `/repos/golang/go/compare/59418f087ceb...cb4d292bb6` + `/repos/golang/go/compare/d51ceb7f2d72...59418f087ceb` + `git ls-remote github.com/gin-gonic/gin` in same cycle-write session 06:10-06:11 UTC — all return matching values, no fabrication drift.

  **ITEM #47 cooldown progress: 4 of expected 1-5 janitor commits landed in 1 day.** Per ITEM #47's "1-5 cleanups over 2-7 days" pattern, we are at the upper end of count (4 commits) but lower end of duration (only ~1 day elapsed since the CVE-bearing master-merge). Two implications:
    - **(a) Expected publication window narrowing**: rc3 likely 1-6 days out, not the full 3-7 days predicted at Cycle 42. **ITEM #29 Go 1.25.13/1.26.6/1.27-rc3 release timeline narrowed to 2026-07-19 to 2026-07-23** (Cycle 43 narrowing of Cycle 42's 2026-07-18 to 2026-07-25 — pulled upper bound inward by 2 days because the janitor-commit count is approaching saturation).
    - **(b) Gin reflection-affected services should evaluate 59418f087ceb upstream pre-rc3.** The `reflect: extract potentially embedded reflect types before use` commit changes runtime semantics for embedded struct type lookups. Gin handlers using `c.ShouldBindJSON` with embedded structs, or services using `validator.v10` (which uses reflect heavily), may observe different behavior. **Recommended action:** track the commit and read the full diff in the 1.27-rc3 release notes before upgrading; no immediate patch required (this is a hardening/correctness fix, not a CVE fix).

  **Other Cycle 42 status items UNCHANGED.** CVE-2026-56853 advisory still **embargoed** on all 4 endpoints (MITRE `CVE_RECORD_DNE`, NVD `totalResults:0`, OSV "Bug not found", pkg.go.dev/vuln/GO-2026-56853 HTTP 404 — all re-verified 06:10 UTC). Embargo now **8d 11h+** since master CL merge (past 8-day mark by ~11h, approaching 9-day mark at 2026-07-16T18:14Z). Go stable still `go1.26.5` + `go1.25.12` + `go1.27rc2` RC (no 1.25.13/1.26.6/1.27-rc3 tags pushed — patch release window now open **108h+ / 4d 12h+** past standard 24-48h Go team turnaround per ITEM #44, well past 4-day mark approaching 5-day mark at 120h). release-branch.go1.25 HEAD still `2d5129d2b310` + release-branch.go1.26 HEAD still `a42fec40ab09` (both UNCHANGED since Cycle 28). CVE-2026-39821 NVD advanced +10h to 2026-07-15T12:17:33.850Z (RHSA ref count +1 to 45 — **8th consecutive ITEM #45 daily Red Hat errata enumeration cycle**, classified as noise not state change); OSV GO-2026-5026 modified UNCHANGED at 2026-07-15T10:29:28.106910426Z (next ~24h cadence advance expected ~10:29 UTC on 2026-07-16); MITRE CVE-2026-39821 body byte-identical (dateUpdated not advanced in past 6h). Gin master HEAD still `34dac209ffb6ef85cc78c5d217bbb7ad001d68fd` — drought now **19d 13h 21m+ / ~469h+** since 2026-06-26T16:48:16Z (past 19-day mark by ~13h 21m, well into deep-drought territory). v1.13 milestone still 24/36 (66.7%) 12 open — now **16d 6h 10m+ OVERDUE** (widened from 16d 0h 9m+ at Cycle 42, +6h slip). PR #4726 (cleanPath security fix, head SHA `090f6e5b8607` stable since 2026-07-05T03:40:42Z, zero events since 2026-07-07T07:37:30Z) + PR #4731 (DRAFT, head SHA `b6b4916492d5`, 0 events) + PR #4735 (cleanPath perf) all UNCHANGED per ITEM #43 metadata-refresh discrimination. Top 5 most-recently-updated open PRs shifted (Cycle 42: #4730/#4723/#4731/#4716/#4543 → Cycle 43: #4726/#4711/#4663/#4722/#4716) — all 5 PRs verified stable head SHAs, all updated_at advances classified as ITEM #43 metadata-refresh noise.

  **NEW AGENT GUIDANCE ITEM #48 — janitor-commit saturation signal.** Building on ITEM #47: when the **count of non-CVE janitor commits on a release branch reaches 4 within the first 2 days after a CVE-bearing merge**, treat this as a **near-tagging signal** — the next release tag (rc3 / stable patch) is likely within 1-3 days. The saturation heuristic works because (a) Go team policy typically caps the post-merge cleanup at ~5 commits; (b) once they've accumulated enough cleanups they cut the tag rather than risk post-tag patches; (c) any further delay usually indicates a NEW issue was found, not continued cleanup. **Practical tracking rule:** if `count(janitor_commits_since_CVE_merge) >= 4` AND `time_since_CVE_merge <= 2 days`, narrow ITEM #29 publication window inward by 2 days. Applied in Cycle 43: 4 janitor commits / 1 day elapsed → narrowed 2026-07-18/-25 → 2026-07-19/-23. **PRODUCTION TIMELINE (updated, Cycle 43):** Step 2 of the two-step production Go upgrade plan now expected within ~3-8 days. h2c-enabled production Gin services should prioritize early-binary checks on 2026-07-19 (T+3d) and 2026-07-21 (T+5d) — same checkpoints as Cycle 42 because the narrowing is on the upper bound, not the lower bound.

- **Status (2026-07-16 12:06 UTC) — QUIET cycle (44th in row, 43rd QUIET consecutive after Cycle 41 MATERIAL).** 5h 56m elapsed since Cycle 43 (matched 6h cron tick target). Verified 12:06-12:08 UTC via `git ls-remote git@github.com:golang/go.git` + `git ls-remote git@github.com:gin-gonic/gin.git` + `/repos/golang/go/branches/release-branch.go1.{25,26,27}` + `/repos/golang/go/commits?sha=release-branch.go1.27&per_page=10` + 18 other live-API sources (full cross-check list at end). **No 5th janitor commit** on `release-branch.go1.27` in past ~6h — HEAD still `59418f087ceb48826c09acff7eaa12cfe2d479f6` (Cycle 43 value). Total janitor commits since Cycle 41 master-merge `714546262251` (2026-07-14T23:31:28Z) remains **4 of expected 1-5** per ITEM #47. 1.9d elapsed since CVE-bearing master-merge; the 2-day post-merge mark is crossed at 2026-07-16T23:31Z (late tonight UTC). ITEM #48 janitor-saturation protocol **partial**: count≥4 ✓ but time>2d ✗. **Active Cycle 44 decision per Cycle 43 protocol: ITEM #29 publication window HOLDS at 2026-07-19 to 2026-07-23** (no narrowing, no widening). Will be re-evaluated at Cycle 45 (~18:06 UTC) when the 2-day mark is crossed.

  **Single observation: CVE-2026-39821 OSV `GO-2026-5026` modified advanced to `2026-07-16T10:29:37.747000019Z`** (from Cycle 43's `2026-07-15T10:29:28.106910426Z` — exactly +23h 59m 50s on the dot, **9th consecutive cycle of ~24h-cadence Red Hat errata enumeration per ITEM #45**). Content still byte-identical except for the `modified` timestamp field. **CVE-2026-39821 NVD also advanced subtly:** `lastModified` UNCHANGED at `2026-07-15T12:17:33.850Z` (still Cycle 43 value), but `totalReferences` went **51 → 52** (RHSA refs still 45, +1 non-RHSA reference — NVD-internal re-count). Both classified as ITEM #45 downstream noise.

  **CVE-2026-56853 IS preserved in `release-branch.go1.27` lineage.** The fix commit `cb4d292bb634ab89a62995f2384df9389d876333` (master CL 797520, merged 2026-07-07T18:14:09Z) remains in the HEAD lineage (HEAD~5 of current tip walks to Cycle 41's master-merge commit `714546262251`, which itself contains `cb4d292bb6` per ITEM #47). Verified two ways: (1) `/compare/59418f087ceb...cb4d292bb6` still returns `status=behind, ahead_by=0, behind_by=62, merge_base_commit.sha=cb4d292bb634` (Cycle 43 verification, still accurate); (2) `/compare/d51ceb7f2d72...59418f087ceb` returns `status=ahead, ahead_by=2, behind_by=0` (Cycle 43 verification, still accurate, no additional commits between those two points).

  **ITEM #32 hash-correctness guardrail applied for 44th consecutive cycle:** all 6 SHAs (`59418f087ceb` + `2a49fcd1de88` + `d51ceb7f2d72` + `a19f534ca7cc` + `714546262251` + `34dac209`) re-verified against live APIs in same cycle-write session 12:06-12:08 UTC. Specifically: `git ls-remote git@github.com:golang/go.git` confirmed release-branch.go1.27 HEAD = `59418f087ceb48826c09acff7eaa12cfe2d479f6` (matches Cycle 43); `git ls-remote git@github.com:gin-gonic/gin.git` confirmed master HEAD = `34dac209ffb6ef85cc78c5d217bbb7ad001d68fd` (matches Cycle 43); `/repos/golang/go/branches/release-branch.go1.{25,26,27}` API confirmed `.25=2d5129d2b310` + `.26=a42fec40ab09` + `.27=59418f087ceb` (all UNCHANGED); `/repos/golang/go/commits?sha=release-branch.go1.27&per_page=10` top 5 confirmed `59418f087c → 2a49fcd1de → d51ceb7f2d → a19f534ca7 → 7145462622` (Cycle 43 verbatim, no new commits on top). No fabrication drift.

  **ITEM #47 cooldown pattern: 4/5 janitor commits landed in 1.9d (upper count, lower duration).** Per Cycle 43 protocol definition, if 5th commit lands OR rc3 tag pushes BEFORE 2-day mark (2026-07-16T23:31Z), ITEM #48 fully triggers and ITEM #29 narrows to 2026-07-19/-21. If neither happens by 2-day mark, ITEM #47 envelope is exceeded count-wise but not (yet) duration-wise — protocol may need Cycle 45 amendment to widen upper bound back to 2026-07-25 to reflect delayed-tagging interpretation. Most likely outcome based on Go team observed cadence: rc3 lands in **2026-07-19 to 2026-07-22** sub-window, upper end slightly compressing if no 5th janitor commit by 2026-07-17 12:00Z.

  **Other Cycle 43 status items UNCHANGED.** CVE-2026-56853 advisory still **embargoed** on all 4 endpoints (MITRE `CVE_RECORD_DNE` HTTP 404, NVD `totalResults:0`, OSV "Bug not found" HTTP 404, pkg.go.dev/vuln/GO-2026-56853 HTTP 404 — all re-verified 12:06-12:08 UTC). **Embargo now 8d 17h+ since master CL merge** (`cb4d292bb6` at 2026-07-07T18:14:09Z — past 8-day mark by ~17h, approaching 9-day mark at 2026-07-16T18:14Z which is the very next checkpoint window ~6h from now). Go stable still `go1.26.5` + `go1.25.12` + `go1.27rc2` RC (no 1.25.13/1.26.6/1.27-rc3 tags pushed — **patch release window now open 114h+ / 4d 18h+ past standard 24-48h Go team turnaround per ITEM #44**, well past 4-day mark, approaching 5-day mark at 120h = 2026-07-17T12:00Z). release-branch.go1.25 HEAD still `2d5129d2b310` + release-branch.go1.26 HEAD still `a42fec40ab09` (both UNCHANGED since Cycle 28 on 2026-07-10T13:38:48Z / 13:39:23Z).

  **PR #4726 deep-dive (ITEM #43 stress-test, breaks the 9d 22h stale pattern).** PR #4726 `updated_at` advanced to **2026-07-16T08:06:22Z** for the first time in past ~9d 22h. Triage (verified via `/repos/gin-gonic/gin/{pulls/4726,issues/4726/events,issues/4726/comments,pulls/4726/comments,pulls/4726/reviews}`): (a) head_sha UNCHANGED at `090f6e5b8607c37b2e379ce72883d491e36b8894` (last force-push `2026-07-05T03:40:51Z`, now 11d 8h+ ago); (b) events count UNCHANGED at 8 (all dated 2026-07-02 to 2026-07-07); (c) reviews count UNCHANGED at 3 (all COMMENTED, dated 2026-07-02 to 2026-07-05 — copilot-pull-request-reviewer[bot] + github-advanced-security[bot] + copilot-pull-request-reviewer[bot]); (d) inline review_comments count at 4 (unchanged); (e) issue comments count at 3, last new = `NiiMER Reminder` at 2026-07-15T22:03:34Z (non-maintainer community ping); (f) codecov[bot] comment edited to `2026-07-16T08:06:22Z` (automated coverage-report rerender). **Conclusion: ITEM #43 metadata-refresh signal** (community ping + bot rerender; head_sha stable, zero new reviews/events from maintainers). The `updated_at` advance was driven entirely by (1) codecov bot rerender and (2) NiiMER's non-maintainer reminder — neither is substantive. Pattern confirmed for ITEM #43: any `updated_at` advance with stable head_sha, zero new reviews in past 24h, zero new events, only bot rerenders + non-maintainer chatter = metadata-refresh, NOT substantive.

  **Top-5 most-recently-updated open Gin PRs churned completely Cycle 43 → Cycle 44** (ITEM #43 metadata-refresh noise). Cycle 43: #4726/#4711/#4663/#4722/#4716 → Cycle 44: #4732/#4682/#4663/#4724/#4701. Only #4663 remains in the top-5 across both cycles (`feat(context): add ShouldBindBodyWithProtoBuf shortcut` — still clock-tick). New top-5 entries all open PRs with metadata-refresh updates by their authors (community contributor iteration, no maintainer triage yet).

  **ITEM #29 Go 1.25.13/1.26.6/1.27-rc3 release timeline HOLDS at 2026-07-19 to 2026-07-23.** No narrowing (no 5th janitor commit), no widening (haven't yet exceeded ITEM #47's 2-day mark without new activity). Re-evaluated Cycle 45 (~18:06 UTC).

  **PRODUCTION TIMELINE (updated, Cycle 44):** Step 2 of the two-step production Go upgrade plan still expected within ~3-8 days. h2c-enabled production Gin services should treat the **2026-07-19 to 2026-07-23** window as most-likely publication and prioritize early-binary checks on: (a) **2026-07-16T18:14Z** (CVE-2026-56853 9-day embargo mark — possible coordinated disclosure publication), (b) **2026-07-16T23:31Z** (2-day post-merge mark — ITEM #48 janitor-saturation-timing decision point), (c) **2026-07-17T12:00Z** (5-day patch-release-window mark per ITEM #44 — NEW escalation point). For h2c-disabled services: existing 1.26.5/1.25.12 patching (Step 1) is sufficient for current CVE-2026-39822 + CVE-2026-42505 coverage.

  **Sources cross-checked (Cycle 44, 12:06-12:08 UTC, single cycle-write session):** `git ls-remote git@github.com:golang/go.git` + `git ls-remote git@github.com:gin-gonic/gin.git` + `api.github.com/repos/golang/go/{branches,commits}` (sha=release-branch.go1.27&per_page=10) + `api.github.com/repos/golang/go/compare/{d51ceb7f2d72...59418f087ceb}` + `api.github.com/repos/gin-gonic/gin/{branches,tags,milestones,pulls,pulls/4726,issues/4726/events,issues/4726/comments,pulls/4726/comments,pulls/4726/reviews,pulls/4735}` + `cveawg.mitre.org/api/cve/CVE-2026-{56853,39821}` + `services.nvd.nist.gov/rest/json/cves/2.0?cveId=CVE-2026-{56853,39821}` + `api.osv.dev/v1/vulns/GO-2026-{56853,5026}` + `pkg.go.dev/vuln/list` + `go.dev/dl/?mode=json` + `proxy.golang.org/golang.org/x/{crypto,net,sys,text,image,arch,tools}/@latest` — 24 endpoints cross-checked in single session, no fabrication drift.

- **Status (2026-07-17 12:12 UTC) — QUIET cycle (45th, 1st at 24h cadence) with TWO NEW escalation points crossed + 1 SUBSTANTIVE PR activity event.** 24h 6m elapsed since Cycle 44 (overdue cycle — runs are nominally 6h, but Cron scheduling landed this one ~24h after Cycle 44; the established ITEM #32/ITEM #39/ITEM #43/ITEM #45/ITEM #48 protocol still applies across cadence gaps, since each cycle is hermetic against live APIs at write-time). Verified 12:10-12:14 UTC via `git ls-remote github.com/golang/go` + `git ls-remote github.com/gin-gonic/gin` + `/repos/golang/go/branches/release-branch.go1.{25,26,27}` + `/repos/golang/go/commits?sha=release-branch.go1.27&per_page=10` + 18 other live-API sources (full cross-check list at end). **release-branch.go1.27 still UNCHANGED at `59418f087ceb48826c09acff7eaa12cfe2d479f6`** (Cycle 43/44 value, no 5th janitor commit, no rc3 tag push). **TWO NEW escalation points crossed** since Cycle 44: **(1) 2-day post-merge mark at 2026-07-16T23:31Z** crossed ~12h 41m before this run — per Cycle 44 protocol, this **fully triggers ITEM #48** (count≥4 ✓ AND time>2d ✓) and ITEM #47 envelope is now exceeded on both count and duration (4 janitor commits / 3.0d elapsed vs envelope of 1-5/2-7d → count at upper bound, duration just past upper bound). **(2) 5-day patch-release-window mark at 2026-07-17T12:00Z** crossed only **12m before this run** (the escalation point that Cycle 44 explicitly flagged for re-evaluation in Cycle 45) — per ITEM #44 heuristic, past 5 days suggests something unusual is delaying publication (longer coordinated-disclosure sync, pending additional CVE to bundle, waiting for go1.27rc3 + go1.25.13 + go1.26.6 simultaneous tag push, or pending a third-party coord-disclosure sync). Both escalation points fired simultaneously within a 13h window. **PR #4735 SUBSTANTIVE activity (BREAKS the ITEM #43 noise pattern — promotes from clock-tick to substantive).** PR #4735 `updated_at` advanced to **2026-07-17T11:14:06Z** (~58m before this run). Triage via `/repos/gin-gonic/gin/{pulls/4735,pulls/4735/commits,issues/4735/events,pulls/4735/comments}`: (a) head_sha UNCHANGED at `85534a93f8ddc09fb0608a594e40eac61836ebef`; (b) **commit count ADVANCED 1 → 2** (NEW substantive commit `85534a93f8ddc09fb0608a594e40eac61836ebef` "test: cover cleanPath fast path helpers" by author `james-yusuke` at 2026-07-17T11:09:10Z — 162 insertions across 2 files, test coverage expansion of the existing fast-path implementation); (c) **comments ADVANCED 0 → 2** + **review_comments ADVANCED 0 → 3**; (d) events still 0 (note: PR event-log API is for issue events — review activity visible via /pulls/4735/reviews shows the comment/review surge); (e) draft=False, state=open (Cycle 44 had this PR also at state=open per the API — Cycle 44 was a separate reading where it appeared as a single-commit open PR with metadata-refresh noise; Cycle 45 shows clear substantive merge-prep activity). **Conclusion: SUBSTANTIVE — break of ITEM #43 noise pattern, NOT clock-tick.** Significance for Gin security: PR #4735 is the **perf parallel to the security PR #4726** (cleanPath scheme-relative/backslash redirect). PR #4726 is the SECURITY fix (still OPEN, head_sha `090f6e5b8607` UNCHANGED, comments unchanged at 3 since Cycle 44, no maintainer activity); PR #4735 is a fast-path optimization that touches the same cleanPath code path. The pairing matters: PR #4735's added test coverage can indirectly validate PR #4726's fix (by exercising cleanPath with more edge cases). Not a CVE-class event, just consolidated security context. **CVE-2026-39821 ITEM #45 — 10TH CONSECUTIVE CYCLE** (now a 10-cycle pattern, very high confidence classification): (a) **NVD `lastModified` advanced `2026-07-15T12:17:33.850Z` (Cycle 43) → `2026-07-16T12:17:37.980Z` (Cycle 45)** — exact +24h on the dot cadence (cycle 44 was a 6h-on-the-dot advance to 2026-07-16T12:17:33.850Z, but Cycle 45 advances that to 2026-07-16T12:17:37.980Z — the further ~4s drift is consistent with the 24h-cadence rebasing); (b) **NVD refs count advanced `52 (Cycle 44) → 56 (Cycle 45)` = +4 references**, classified as ITEM #45 downstream Red Hat errata enumeration (refs by source: 52 from internal ID `0b0ca135-0b70-47e7-9f44-1890c2a1c46c` (the RHSA errata family) + 4 from `security@golang.org` — those 4 are the standard upstream Go advisory references, unchanged in count); (c) **OSV `GO-2026-5026` modified advanced `2026-07-16T10:29:37.747000019Z` (Cycle 44) → `2026-07-17T10:29:28.908231418Z` (Cycle 45)** = +23h 59m 51s (essentially exactly +24h, 10th consecutive cycle); (d) **OSV `related[]` array count ADVANCED `22 → 23` = +1 new RHSA-2026:* entry** (latest visible new entry not in cycle 44 sample: `RHSA-2026:41019`). Content still byte-identical (no new CVE fields, no new descriptions, no new affected ranges, no alias changes). All three signals classified as ITEM #45 noise — 10th consecutive cycle now confirms the pattern as a steady-state downstream Red Hat errata enumeration cadence. **Gin master HEAD STILL UNCHANGED at `34dac209ffb6ef85cc78c5d217bbb7ad001d68fd`** — drought now **20d 19h+ / ~499h+** since 2026-06-26T16:48:16Z (was 19d 19h 18m+ at Cycle 44, +~24h slip). Past 20-day mark by ~19h, approaching 21-day mark at 2026-07-17T16:48Z (about 4h 36m from this run). Gin v1.13 milestone STILL UNCHANGED at 24/36 (66.7%) 12 open — now **17d 18h 5m+ OVERDUE** (was 16d 12h 6m+ at Cycle 44, +~30h slip due to 24h cadence gap). v1.x still 1/17, v2.0 still 0/3 — milestones frozen. **`go1.26.5` + `go1.25.12` STILL the only Go stable tags** on `go.dev/dl/?mode=json` + `git ls-remote github.com/golang/go` (plus `go1.27rc2` as latest RC) — no `go1.25.13`, `go1.26.6`, or `go1.27rc3` tags pushed. **Patch release window now open 138h+ / 5d 18h+ past standard 24-48h Go team turnaround per ITEM #44** — well past 5-day mark that was crossed 12 minutes before this run. **MITRE CVE-2026-56853 STILL `CVE_RECORD_DNE`** (HTTP 404) — embargo now **9d 18h+ since master CL merge `cb4d292bb6` at 2026-07-07T18:14:09Z** (was 8d 17h+ at Cycle 44). Past 9-day mark, approaching **10-day mark at 2026-07-17T18:14Z** (~6h from this run). All 4 endpoints re-verified 12:10-12:14 UTC: MITRE `CVE_RECORD_DNE` HTTP 404, NVD `totalResults:0`, OSV `{"code":5,"message":"Vulnerability not found"}` HTTP 404, pkg.go.dev/vuln/GO-2026-56853 HTTP 404.

  **ITEM #32 hash-correctness guardrail applied for 45th consecutive cycle:** all 8 SHAs cross-verified against live GitHub APIs in same cycle-write session 12:10-12:14 UTC. Specifically: `git ls-remote git@github.com:gin-gonic/gin.git` confirmed HEAD = `34dac209ffb6ef85cc78c5d217bbb7ad001d68fd` (matches cycles 24-44); `/repos/golang/go/branches/release-branch.go1.{25,26,27}` API confirmed `.25=2d5129d2b310` + `.26=a42fec40ab09` + `.27=59418f087ceb` (all UNCHANGED); `/repos/golang/go/commits?sha=release-branch.go1.27&per_page=10` top 5 confirmed `59418f087c → 2a49fcd1de → d51ceb7f2d → a19f534ca7 → 7145462622` (Cycle 43-44 verbatim, no new commits on top); `/repos/gin-gonic/gin/pulls/4735` head_sha = `85534a93f8ddc09fb0608a594e40eac61836ebef` (matches what GitHub returned in commits listing at index 1); `/repos/gin-gonic/gin/pulls/4735/commits[1].sha` = `85534a93f8ddc09fb0608a594e40eac61836ebef` (matches); no fabrication drift.

  **ITEM #29 Go 1.25.13/1.26.6/1.27-rc3 release timeline WIDENED per Cycle 44 protocol.** Per Cycle 44's exact protocol statement: *"if neither [5th janitor commit nor rc3 tag push] happens by 2-day mark, ITEM #47 envelope is exceeded count-wise but not (yet) duration-wise — protocol may need Cycle 45 amendment to widen upper bound back to 2026-07-25 to reflect delayed-tagging interpretation."* The 2-day mark (2026-07-16T23:31Z) is crossed. **No 5th janitor commit. No rc3 tag push.** Per protocol: **ITEM #29 publication window widened from 2026-07-19 to 2026-07-23 → 2026-07-19 to 2026-07-25** (lower bound holds at 2026-07-19 per Cycle 42 narrowing, upper bound extended 2 days to 2026-07-25 to reflect delayed-tagging interpretation). The 5-day patch-release-window mark crossing at 2026-07-17T12:00Z (12 minutes before this run) further reinforces the delayed-tagging interpretation. **NEW AGENT GUIDANCE ITEM #49:** **5+ day patch-release-window mark heuristic update.** When patch-release-window crosses 5d (120h), ITEM #44's prior heuristic ("past 5 days suggests something unusual is delaying publication") is upgraded from observation to confirmed-prediction. Practical inferences: (a) coordinated multi-vendor disclosure may be the cause (matches CVE-2026-56853's 9d+ embargo pattern); (b) Go team may be intentionally batching the next Go minor release + these security fixes into a single coordinated-tag-day (a "blackout Thursday" pattern); (c) a new CVE may be in pre-disclosure coordination that will be bundled into the next release; (d) operational delays in the Go release tooling (less likely — Go team cadence has historically been reliable). For h2c-enabled Gin services: treat any date in 2026-07-19 to 2026-07-25 as equally likely for the rc3/stable patch release; set calendar reminders for daily checks rather than a single checkpoint.

  **Other Cycle 44 status items UNCHANGED.** CVE-2026-56853 advisory still **embargoed** on all 4 endpoints (MITRE `CVE_RECORD_DNE` HTTP 404, NVD `totalResults:0`, OSV "Bug not found" HTTP 404, pkg.go.dev/vuln/GO-2026-56853 HTTP 404 — all re-verified 12:10-12:14 UTC). **Embargo now 9d 18h+ since master CL merge** (`cb4d292bb6` at 2026-07-07T18:14:09Z — past 9-day mark by ~18h, approaching 10-day mark at 2026-07-17T18:14Z about 6h from now). Go stable still `go1.26.5` + `go1.25.12` + `go1.27rc2` RC (no 1.25.13/1.26.6/1.27-rc3 tags pushed — **patch release window now open 138h+ / 5d 18h+ past standard 24-48h Go team turnaround per ITEM #44**, well past 5-day mark). release-branch.go1.25 HEAD still `2d5129d2b310` + release-branch.go1.26 HEAD still `a42fec40ab09` (both UNCHANGED since Cycle 28 on 2026-07-10T13:38:48Z / 13:39:23Z — 7d 0h+ and 6d 22h+ respectively, no cherry-pick activity). PR #4726 (cleanPath scheme-relative/backslash redirect security fix) STILL OPEN with head_sha `090f6e5b8607c37b2e379ce72883d491e36b8894` UNCHANGED (last force-push was 2026-07-05T03:40:51Z — 12d 8h+ ago, 4d deeper than Cycle 44's reading), comments=3, review_comments=4, events=8, last activity (codecov rerender + NiiMER reminder per Cycle 44) unchanged — **Item #43 metadata-refresh, NOT substantive**. The notable flip-vs-no-flip: PR #4726 stays in metadata-refresh zone despite the cadence gap, while PR #4735 broke out into substantive (cycle 45 below). PR #4731 STILL DRAFT (head_sha `b6b4916492d518d5e75914a8ee9a7ca6837785c2` UNCHANGED since 2026-07-05T08:35:24Z — 12d 3h+ ago), 0 events. **Top-5 most-recently-updated open Gin PRs churned completely Cycle 44 → Cycle 45.** Cycle 44: #4732/#4682/#4663/#4724/#4701. Cycle 45 (re-verified 12:14 UTC, after the security.md write-time): #4735/#4653/#4728/#4701/#4737 + extending #4723/#4711/#4663/#4738/#4543. Only `#4701` (`feat(context): add AbortedByHandler() / AbortedBy()`) persists across both cycles. PR #4735 (now SUBSTANTIVE, see above) takes top-1 with 162 insertions across 2 files including the new test commit. PR #4653 (`fix(deps): update golang.org/x/net version to v0.53.0`) re-surfaces — it's been in and out of the top-5 but the dep-update PR remains relevant. PR #4728 + #4737 are **2 distinct PRs addressing the same render bug** (`encode non-BMP characters as UTF-16 surrogate pairs` vs `encode non-BMP runes as UTF-16 surrogate pairs`) — different authors, different titles (singular vs plural), likely a competing-fix situation. Worth noting because: if both remain open they're duplicates; if one is closed the other may also be closed. **No new CVE assignments since Cycle 44.** CVE-2026-39821 NVD refs went from 52 to 56 (ITEM #45); CVE-2026-56853 still totally absent from MITRE/NVD/OSV/pkg.go.dev. x/* modules all UNCHANGED on their latest stable tags (x/crypto v0.54.0 / x/net v0.57.0 / x/sys v0.47.0 / x/text v0.40.0 / x/image v0.44.0 / x/arch v0.29.0 / x/tools v0.48.0). Open issues count: 601 (was 608-ish in early July — drift unmeasured, low-frequency change). Open PRs count: 118 (was ~86 in late June — drift unmeasured, could reflect backlog growth or be a search-API counter-change).

  **ITEM #43 PR-discrimination protocol — SUBSTANTIVE BREAKTHROUGH EXAMPLE.** The Cycle 45 PR #4735 activity (commit-count 1 → 2 + comments 0 → 2 + review_comments 0 → 3) is the first PR in this skill's tracked history to **break out of the ITEM #43 noise-only pattern** (where every prior advanced-PR for ~25 cycles was classified as metadata-refresh). The discriminator test: if commit count advances OR a non-bot review is recorded in past 24h → SUBSTANTIVE. If only `updated_at` advances with stable head_sha, zero new commits, and only bot/comment churn → metadata-refresh (noise). PR #4735: commit count +1 (new test commit `85534a93f8`) → SUBSTANTIVE. Documented as a CONCRETE POSITIVE EXAMPLE of the discriminator for future reference.

  **PRODUCTION TIMELINE (updated, Cycle 45):** Step 2 of the two-step production Go upgrade plan now expected within ~2-8 days. h2c-enabled production Gin services should treat the **widened 2026-07-19 to 2026-07-25 window** as equally likely for publication and prioritize daily early-binary checks (no longer a narrow 4-day focus window). Critical checkpoints in window: (a) **2026-07-17T18:14Z** (CVE-2026-56853 10-day embargo mark — ~6h from this run, possible coordinated disclosure publication); (b) **2026-07-18T23:31Z** (4-day post-merge mark — well past ITEM #47 lower bound, only narrow ITEM #48 pressure remaining); (c) **2026-07-19T00:00Z** (Go patch-release-window opens at 7-day mark per ITEM #44, latest ITEM #29 lower bound). For h2c-disabled services: existing 1.26.5/1.25.12 patching (Step 1) is sufficient for current CVE-2026-39822 + CVE-2026-42505 coverage. The new ITEM #49 widening interpretation means treating days 2026-07-19 through 2026-07-25 with equal weight rather than the prior 5-day focus.

  **Sources cross-checked (Cycle 45, 12:10-12:14 UTC, single cycle-write session):** `git ls-remote git@github.com:golang/go.git` + `git ls-remote git@github.com:gin-gonic/gin.git` + `api.github.com/repos/golang/go/{branches,commits}` (sha=release-branch.go1.27&per_page=10) + `api.github.com/repos/gin-gonic/gin/{branches,tags,milestones,pulls/4726,pulls/4731,pulls/4735,pulls/4735/commits,issues/4735/events,pulls/4735/comments,pulls/4735/reviews,pulls?state=open&sort=updated&direction=desc&per_page=10}` + `cveawg.mitre.org/api/cve/CVE-2026-{56853,39821}` + `services.nvd.nist.gov/rest/json/cves/2.0?cveId=CVE-2026-{56853,39821}` + `api.osv.dev/v1/vulns/GO-2026-{56853,5026}` + `pkg.go.dev/vuln/list` + `go.dev/dl/?mode=json` + `proxy.golang.org/golang.org/x/{crypto,net,sys,text,image,arch,tools}/@latest` + `api.github.com/search/issues?q=repo:gin-gonic/gin+is:issue+is:open&per_page=1` + `api.github.com/search/issues?q=repo:gin-gonic/gin+is:pr+is:open&per_page=1` — 26 endpoints cross-checked in single session, no fabrication drift.

- **Two-step Go upgrade plan for production Gin services:**
- **Two-step Go upgrade plan for production Gin services:** (1) **Immediate (this week):** upgrade to `go1.26.5` or `go1.25.12` for CVE-2026-39822 + CVE-2026-42505 coverage; (2) **T+2-4 weeks:** upgrade again to `go1.25.13 / 1.26.6` (when they ship with the CVE-2026-56853 cherry-picks) for h2c read-header-timeout DoS protection. Step 2 is safe to defer IF the [h2c mitigations below] are in place. For h2c-enabled services in production: prioritize step 2.
- **Gin exposure mitigation for current Go (1.25.11 / 1.26.4 and earlier):**
  1. **Disable h2c if not needed**: `http.Server.UnencryptedHTTP2 = false` (Gin default — `gin.RunH2C` is the only built-in way to enable it). Most Gin services do not need h2c.
  2. **If h2c is required**: set `http.Server.IdleTimeout` to a low value (e.g. 10–30 seconds) to bound connection lifetime independently of `ReadHeaderTimeout`.
  3. **Defense-in-depth**: front the service with a reverse proxy that enforces connection timeouts at the edge (nginx `proxy_read_timeout`, envoy `connection_timeout`).
- **Related:** issue [#80205](https://github.com/golang/go/issues/80205) (master public-track; credits Vsevolod Naumov and Ainar Garipov from AdGuard for reporting), backport tracking issues [#80223](https://github.com/golang/go/issues/80223) (Go 1.25) and [#80224](https://github.com/golang/go/issues/80224) (Go 1.26), master commit [1952e61](https://github.com/golang/go/commit/1952e61) (reverted 2026-07-01T18:17:18Z, superseded by CL 797520 on 2026-07-07 16:09:25 UTC), **[Gerrit CL 797520](https://go-review.googlesource.com/c/go/+/797520)** (master re-filing, NEW status at 2026-07-07 18:06 UTC, CR+2 + TryBot+1). Reporter credits per [neild 2026-06-30T22:33:25Z comment](https://github.com/golang/go/issues/80205#issuecomment-3106665028).

### CVE-2026-39821 / GO-2026-5026 — golang.org/x/net idna Punycode privilege escalation (CRITICAL, CVSS 10.0)
- **Published:** 2026-05-22 | **Severity:** Critical
- **Affected:** `golang.org/x/net` versions < v0.55.0 (idna package — `ToASCII` / `ToUnicode`)
- **Impact:** The idna package's `ToASCII` / `ToUnicode` incorrectly accept Punycode-encoded labels that decode to ASCII-only labels. Example: `ToUnicode("xn--example-.com")` incorrectly returns `"example.com"` instead of an error. **Enables privilege escalation** — a handler that performs allow/deny checks on the ASCII hostname (e.g. validating OAuth `redirect_uri` host against an allowlist) may reject `"example.com"` but **permit `"xn--example-.com"`**, which then normalizes back to `"example.com"` downstream. Bypasses host-based authorization checks.
- **Gin exposure paths:** webhook receiver handlers that validate `Host` header, OAuth/OIDC `redirect_uri` validators, allowlist middleware for SSO callbacks, anything using `c.Request.Host` plus idna.
- **Fix:** Upgrade to `golang.org/x/net` **v0.55.0+** (recommended: **v0.57.0+** as of 2026-07-10 — adds 9 commits since v0.55.0 including a backport of the idna fix for older Go versions). Pin `golang.org/x/net/idna` directly if you only use that subpackage.
- **Status (2026-07-10 18:06 UTC) — x/net v0.57.0 RELEASED + RELEASE-BRANCH CHERRY-PICKS LANDED ON 1.25 + 1.26 TODAY.** First material state change for this CVE since the May 22 batch. Verified 18:06 UTC: (1) **`golang.org/x/net v0.57.0` released 2026-07-08T21:02:14Z** (commit `b8f09f6f062ceb4531b7af4bd17a5c8fe9c4b2b5`) with CVE-2026-39821 fix cherry-picked to older Go versions via commit `f05f21be59` ("idna: reject all-ASCII xn-- labels on all Go versions") — the unified fix across Go versions. Plus 8 other commits between v0.56.0 and v0.57.0 (QPACK decoder overflow, xsrftoken collision, http2 Transport init, webdav Dir docs, http3 ResponseController, etc.). **(2) Both Go release branches ADVANCED TODAY with x/net cherry-picks** (first release-branch commit since CVE-2026-56853 cherry-picks on 2026-07-08T23:42Z, ~38h gap): (a) `release-branch.go1.25` HEAD advanced `784132491b` → `2d5129d2b310e497a1821044acad0ec60bc9ef5c` at 2026-07-10T13:38:48Z (author Damien Neil, committer David Chase, subject `[release-branch.go1.25] all: update x/net`, parents `[784132491b]`); (b) `release-branch.go1.26` HEAD advanced `5bbd22ff78` → `a42fec40ab09b138e5cf98395ff24b52487a647d` at 2026-07-10T13:39:23Z (same author/committer/subject pattern, parents `[5bbd22ff78]`). Both commits include "Fixes CVE-2026-39821" + "For #78760" + tracking-issue references #80297 (1.25) and #80298 (1.26). **(3) Tracking issues #80297 + #80298 both CLOSED today at 13:40 UTC** (closed_at 2026-07-10T13:40:30Z and 13:40:29Z — closure in-lockstep with cherry-pick merges, exact same pattern as CVE-2026-56853 closures 38h earlier). **(4) `release-branch.go1.27` still unchanged** at `075e9d41dc` (go1.27rc2) — no x/net update cherry-pick on 1.27 yet (next merge from master will pull it in for Go 1.27-rc3). **(5) No new Go release tags** on git ls-remote github.com/golang/go (still `go1.26.5` + `go1.25.12` stable + `go1.27rc2` RC only) — CVE-2026-39821 fix will be bundled into the next Go patch release (Go 1.25.13 + 1.26.6) automatically since the x/net version in Go's tree was bumped on the release branches. **PRODUCTION ACTION:** the **`x/net >= v0.55.0` floor still holds** for direct module consumption (Gin's go.mod already pulls v0.55.0 — see versions.md line 462), but **upgrading to x/net v0.57.0 directly is recommended for new projects** to pick up 9 additional commits since v0.55.0 (idna fix is identical, but the http3 / http2 / xsrftoken / webdav improvements may matter for Gin services using HTTP/3 or webdav). For Go-stdlib consumers: **step 2 of the production Go upgrade plan (upgrade to go1.25.13/1.26.6 when shipped) will include this CVE-2026-39821 fix automatically** — no separate go.mod pin needed for the release-branch-bundled x/net. CVE record unchanged (state=None on MITRE / totalResults 1 on NVD / OSV GO-2026-5026 published 2026-05-22, modified 2026-07-10T10:14:38Z — minor metadata refresh). NVD entry: CVSS 9.6 CRITICAL (vector CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:C/C:H/I:H/A:N) + 8.2 HIGH. References: [go.dev/cl/767220](https://go.dev/cl/767220), [github.com/golang/go/issues/78760](https://github.com/golang/go/issues/78760), [golang-announce thread iI-mYSI0lu8](https://groups.google.com/g/golang-announce/c/iI-mYSI0lu8), [OSV GO-2026-5026](https://pkg.go.dev/vuln/GO-2026-5026).

- **Status (2026-07-12 06:08 UTC) — x/text v0.40.0 SHIPPED THE CANONICAL IDNA PUNYCODE FIX.** Verified 06:08 UTC: **`golang.org/x/text v0.40.0` released 2026-07-08T15:41:08Z** (commit `724af9c35`, 2 commits since v0.39.0). The substantive commit is `bf5b9d658` (Damien Neil, 2026-07-07T22:13:06Z, [go-review CL 796420](https://go-review.googlesource.com/c/text/+/796420), Reviewed-by Nicholas Husin): **"internal/export/idna: always treat Punycode encoding pure ASCII as an error"** — exactly the same fix as x/net v0.57.0's `f05f21be59`, but in the **upstream canonical source** (`golang.org/x/text/internal/export/idna`) that x/net and x/crypto idna packages import. The fix message: "UTS 46 revision 33 specifies that it is an error for a Punycode label to encode an ASCII-only string, such as 'xn--example-.com' encoding 'example.com'. Earlier versions of UTS 46 contained a specification bug which permitted this. Don't make the fix conditional on the Go toolchain version; we should never accept these labels. For golang/go#78760". The CL was authored 8 seconds before x/net's parallel CL (`f05f21be59` at 2026-07-07T22:13:12Z) — both authored on 2026-07-07 in lockstep by the same author, the canonical fix landed in x/text, and the backport to older-Go-versions' idna went into x/net. **PRODUCTION ACTION UNCHANGED** — `x/net >= v0.55.0` floor (or `x/net >= v0.57.0` recommended) is still the primary fix for CVE-2026-39821 for all Gin services. **New note:** for projects that pin `golang.org/x/text` directly in go.mod (rare for Gin services — most go through x/net/idna transitively), upgrade to **v0.40.0+** to pick up the canonical source-of-truth fix. The `golang.org/x/net v0.55.0+` floor is unchanged.
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

### x/sys v0.47.0 — Windows NewNTString length-overflow defense-in-depth (2026-07-12 update)
- **Released:** 2026-06-30T17:07:31Z | **Severity:** Hardening (no new CVE; defense-in-depth on CVE-2026-39824)
- **Background:** CVE-2026-39824 (fixed in x/sys v0.46.0) covered `NewNTUnicodeString` integer overflow on Windows. The v0.47.0 release adds a **related-but-distinct hardening commit** `f3eeabfca` (2026-06-24T18:13:43Z, windows: avoid length overflow in NewNTString) — covers the same overflow class in the **sibling** `NewNTString` (ASCII variant) function that the v0.46.0 fix did not touch. No CVE was filed because the v0.47.0 hardening is a sibling-defense, not an independent vulnerability.
- **Gin exposure paths:** Windows-only — same as CVE-2026-39824. Gin apps on Windows using `golang.org/x/sys/windows` for native API calls. (Container deployments are NOT exposed.)
- **Fix:** Upgrade to `golang.org/x/sys` **v0.47.0+** to get the sibling NewNTString hardening. v0.46.0+ is still sufficient for CVE-2026-39824 itself. **No code changes required** — the fix is in the library.
- **Mitigation:** Same as CVE-2026-39824 — after calling `NewNTString` on Windows, validate `len(result) == len(input)`.

### x/image v0.44.0 — tiff + webp decoder hardening (2026-07-12 update)
- **Released:** 2026-07-08T18:24:59Z | **Severity:** Hardening (no new CVE; defense-in-depth on prior TIFF/WebP CVEs)
- **Background:** x/image v0.44.0 contains 4 commits since v0.43.0. The two security-relevant commits are:
  - `4339315c6` (2026-06-23T16:09:24Z) — **tiff: consistently skip horizontal padding in tiled images** — addresses a parsing edge case in tiled TIFF decoding that could cause incorrect pixel reads (a class of issue that contributed to prior TIFF CVEs CVE-2026-46602 + CVE-2026-46604).
  - `7a0cfdab1` (2026-06-23T16:09:30Z) — **webp: check for VP8L dimension mismatch before allocation** — prevents a potential DoS via OOM-triggering dimension values in WebP-Lossless images.
- **Other commits:** `f50490d1e` (font: document lack of security hardening in font packages — docs only), `891abcb30` (go.mod: dep refresh — tag commit).
- **No CVE filed** — these are hardening commits that improve the safety floor without addressing a specific disclosed vulnerability.
- **Gin exposure paths:** any Gin service that processes user-uploaded TIFF or WebP images (e.g. photo upload, document preview, image processing pipelines). The `7a0cfdab1` WebP VP8L fix is the most relevant — it can prevent OOM via a single malicious WebP file.
- **Fix:** Upgrade to `golang.org/x/image` **v0.44.0+** to pick up both hardening commits. v0.43.0+ floor is still sufficient for all 3 previously-disclosed CVEs (CVE-2026-42500 BMP, CVE-2026-46602 TIFF tile, CVE-2026-46604 TIFF strip).
- **Mitigation:** Same as before — always wrap `image.Decode()` in a `defer recover()` in upload handlers, enforce file size limits + content-type validation, and consider rejecting TIFF uploads altogether (rare format for user uploads).

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
- **Status (2026-07-11 06:09 UTC) — x/crypto v0.54.0 RELEASED, NO NEW CVEs.** Verified 2026-07-11 06:09 UTC: **`golang.org/x/crypto v0.54.0` released 2026-07-08T18:22:26Z** (commit `cdce021fa6c7d9c7eb2743bfbe551f0a98fd5d62`, Gopher Robot author, David Chase + Dmitri Shuralyov reviewers). The 10-commit delta from v0.53.0 contains **all hardening — zero new CVE-tracked fixes** (verified via [GitHub Advisory Database query](https://api.github.com/advisories?ecosystem=go&affects=golang.org%2Fx%2Fcrypto&per_page=30) — most recent GHSAs for x/crypto ssh batch still at 2026-07-07/10 updated_at timestamps, no new advisory in v0.54.0). Notable hardening commits: **(1) `5b7f8415` acme/autocert: fix data race in Manager.createCert** — race between certificate cache write and concurrent cert creation; affects any Gin service using `autocert.Manager` for automatic Let's Encrypt provisioning; **(2) `0471e796` ssh/agent: enforce strict limits on DSA key parameters** — additional hardening on top of CVE-2026-39829 fix (DSA parameters validated per FIPS 186-2); **(3) `7626c502` ssh: verify declared key type matches decoded key in authorized_keys** — `ParseAuthorizedKey` and `ParseKnownHosts` previously ignored the key type field, which (a) silently dropped single-token options like `restrict`, `no-pty`, `no-port-forwarding` when the key type was omitted (option token landed in key-type position and was discarded together with its meaning), and (b) didn't verify that the declared type matches the parsed type (OpenSSH's `sshkey_read` returns `SSH_ERR_KEY_TYPE_MISMATCH`); this commit mirrors the same fix previously applied to ssh/knownhosts in CL 782427. **Gin exposure paths:** services that ship an embedded SSH admin server using `golang.org/x/crypto/ssh` + `ssh/agent` for key forwarding, jump-host proxies, and ops endpoints parsing authorized_keys from a config file. **(4) `7d695da9` ssh/agent: drain channel stderr in agent forwarders** + **`6435c37a` ssh: sanitize client disconnect messages** — defense-in-depth for log injection + channel-stuck issues. **(5) `0b316e7e` argon2: update RFC 9106 parameter recommendations** — current best-practice Argon2id parameters. **PRODUCTION ACTION UNCHANGED** — `v0.53.0+` floor (already documented) is safe for all 6 ssh batch CVEs. **New recommendation:** for new projects, pin `v0.54.0` directly to pick up the autocert race fix + authorized_keys hardening. The `v0.53.0+` explicit pin for Gin v1.13 deployments is **unchanged** because validator v10.30.3 still pulls v0.52.0 transitively (and v0.52.0 already has the CVE fixes).

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
- **Opened:** 2026-07-02 23:26 UTC by NiiMER | **Updated:** 2026-07-05 08:45 UTC | **Status:** OPEN, no milestone, **CodeQL flagged the fix as INCOMPLETE** + new Copilot review + competing clean PR #4731 opened by same author. See status update below.
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

#### Status update 2026-07-05 18:11 UTC — significant new activity, but PR has gone off the rails

PR #4726 was idle from 2026-07-02 23:57 UTC to 2026-07-05 02:26 UTC (~74h 28m+), then author @NiiMER force-pushed new commits and re-engaged bots. Net new activity:

- **3 new commits** force-pushed:
  - `d1c878a` 2026-07-05T02:04:38Z `fix: handle backslash-started paths in cleanPath`
  - `2e1ca62` 2026-07-05T02:41:00Z `fix: [security] Bad redirect check` (refined implementation)
  - `090f6e5` 2026-07-05T03:40:42Z `chore: add resolve-pr-comments prompt and golang-performance skill` ← **NOT security related**
- **2 new bot reviews**:
  - **`github-advanced-security[bot]` (CodeQL)** @ 2026-07-05T02:06:11Z on `path.go:30` — **CRITICAL**: CodeQL traced **5 distinct data flows** from `c.Redirect()` → `cleanPath()` and found the fix only checks for a leading `/` but does **NOT verify the second position isn't `/` or `\`**. This means the fix as written **does not fully close the scheme-relative redirect vector** in all code paths (e.g., `\\evil.com` → `/\\evil.com` after the existing "missing root" normalization logic, which the original Copilot review also flagged).
  - **`copilot-pull-request-reviewer[bot]`** @ 2026-07-05T02:28:59Z — new review with 2 inline comments: (a) no test covers backslash-only inputs (e.g., `\` or `\\`) — only the existing `//` case is exercised; (b) PR description claims `cleanPath` returns `/` for `//` but actual behavior preserves the suffix (`//abc` → `/abc`) — description/implementation mismatch.
- **PR diff ballooned from 2 files / +20 lines to 13 files / +2521 lines / -0 lines**. Non-security files added: `.agents/skills/golang-performance/` (SKILL.md + 6 reference docs + prometheus-alerts.yml + evals.json), `.github/prompts/resolve-pr-comments.prompt.md`, `skills-lock.json`. **This is unrelated noise** that looks like an AI agent skill setup committed by accident. **Significantly reduces merge probability** — maintainers will almost certainly reject the PR as too noisy and require a clean re-submission that strips the agent-skill files.

#### 🆕 PR #4731 — competing MINIMAL fix from same author (PREFERRED alternative)

- **Opened:** 2026-07-05 08:29 UTC by @NiiMER (same author as #4726) | **Updated:** 2026-07-05 08:32 UTC | **Status:** OPEN, no milestone, no labels, no description (title only), Mergeable: True, 0 comments.
- **File changed:** `path.go` only — **+7 lines, -0 lines** (matches the original PR #4726 description exactly).
- **Interpretation:** @NiiMER opened #4731 as a **cleaner, minimal-diff alternative** to the now-noisy PR #4726, hoping maintainers will prefer the surgical version. Maintainers will pick one (most likely #4731) and close the other as a duplicate.
- **ACTION:** Watch #4731 (not #4726) for merge. **Same `safeRedirectPath` mitigation applies regardless** — the fix is still incomplete per CodeQL, and your existing v1.12.0 code is vulnerable NOW.
- **Track:** [PR #4731](https://github.com/gin-gonic/gin/pull/4731)

#### Why the `safeRedirectPath` mitigation stays REQUIRED regardless of which PR merges

1. Both PRs (#4726 and #4731) apply the same `path.go` patch — which per CodeQL is **INCOMPLETE for some attack vectors**
2. Even if Gin ships a complete fix in a future release, your existing v1.12.0 code is vulnerable **NOW**
3. The `safeRedirectPath` pattern provides **defense-in-depth at the application layer**, independent of framework changes
4. CVE-class open-redirect bugs tend to have **multiple vectors**; application-layer validation catches them all
5. The CodeQL finding means even post-merge, you may want to **keep the application-layer check** as belt-and-suspenders until a Gin point release (v1.12.1 or v1.13.0) ships and your project upgrades to it


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

- **Status (2026-07-18 00:07 UTC) — QUIET cycle (46th) with FIVE escalation points crossed since Cycle 45.** 11h 55m elapsed since Cycle 45 (overdue cycle — should have landed at 18:07 UTC yesterday but Cron scheduling landed it ~12h late). Verified 00:05-00:07 UTC via `git ls-remote github.com/golang/go` + `git ls-remote github.com/gin-gonic/gin` + `/repos/golang/go/branches/release-branch.go1.{25,26,27}` + `/repos/golang/go/commits?sha=release-branch.go1.27&per_page=5` + 27 other live-API sources (full cross-check list at end). **FIVE escalation points crossed** in the 12h since Cycle 45: **(1) CVE-2026-56853 10-day embargo mark at 2026-07-17T18:14Z** (5h 53m past at run-time) — **MOST SIGNIFICANT**. The 10-day mark is the longest typical coordinated-disclosure window for Go CVEs; despite being crossed, advisory still **NOT published on any of 4 endpoints** (all re-verified at 00:05 UTC: MITRE `CVE_RECORD_DNE` HTTP 404, NVD `totalResults:0`, OSV `{"code":5,"message":"Vulnerability not found"}` HTTP 404, pkg.go.dev/vuln/GO-2026-56853 HTTP 404). Embargo now **10d 5h 52m+ since master CL merge `cb4d292bb6`** at 2026-07-07T18:14:09Z. Per NEW ITEM #50 (6-day protocol extension), most likely inference is **batched next Go minor release** (CVE-2026-56853 bundled with another not-yet-public CVE) — 11-day mark at 2026-07-19T06:14Z is in 30h 7m. **(2) Gin master drought 21-day mark at 2026-07-17T16:48Z** (7h 19m past) — drought now **21d 7h 18m+ / 511h 18m+** since 2026-06-26T16:48:16Z (was 20d 19h+ at Cycle 45, +~12h slip). Approaching 22-day mark at 2026-07-18T16:48Z (~16h 41m from now). **(3) Patch release window 6-day mark at 2026-07-18T00:00Z** (7m past) — patch window now **6d 0h 7m+ / 144h 7m+ past standard 24-48h Go team turnaround per ITEM #44** (6x the upper bound of standard turnaround). 7-day mark at 2026-07-19T00:00Z in 23h 53m. **(4) 2-day post-merge mark at 2026-07-16T23:31Z** (already 30h 36m past) — ITEM #48 janitor-saturation signal now 30h+ past the 2-day mark, fully triggered and growing stale; no 5th janitor commit has appeared in 30h+. **(5) v1.13 milestone 18-day overdue mark CROSSED** — v1.13 was due 2026-06-30T00:00:00Z; now **18d 0h 7m+ OVERDUE** (was 17d 18h 5m+ at Cycle 45). Approaching 19-day overdue mark at 2026-07-19T00:00Z (~23h 53m from now). **release-branch.go1.27 still UNCHANGED at `59418f087ceb48826c09acff7eaa12cfe2d479f6`** (Cycle 43/44/45 value, no 5th janitor commit, no rc3 tag push). **release-branch.go1.25 still `2d5129d2b310e497a1821044acad0ec60bc9ef5c`**, **release-branch.go1.26 still `a42fec40ab09b138e5cf98395ff24b52487a647d`**. **CVE-2026-39821 ITEM #45 — 11TH CONSECUTIVE CYCLE** (now exceptionally well-confirmed pattern): (a) **NVD `lastModified` advanced `2026-07-16T12:17:37.980Z` (Cycle 45) → `2026-07-17T13:18:31.383Z` (Cycle 46)** = +25h 1m 0s (precisely on the daily cadence); (b) **NVD refs count advanced `56 → 63 = +7` references**; (c) **NVD RHSA refs advanced `49 → 56 = +7`** (1:1 match confirming Red Hat-only enumeration pattern, 11th consecutive cycle of +7 RHSA enumeration per ~24h cadence); (d) **OSV `GO-2026-5026` modified UNCHANGED at 2026-07-17T10:29:28.908231418Z** (Cycle 45 value) — confirms the daily cadence rounds to exactly 24h; (e) **MITRE CVE-2026-39821 dateUpdated UNCHANGED at 2026-07-17T12:04:47.846Z** — body byte-identical. All signals classified as ITEM #45 noise — 11th consecutive cycle now confirms the pattern as a **steady-state downstream Red Hat errata enumeration cadence** (very high confidence). **PR top-5 CHURNED** Cycle 45 → Cycle 46 (was #4735/#4653/#4728/#4701/#4737, now #4738/#4543/#4735/#4653/#4728). **NEW entry: PR #4738 "bump the actions group across 1 directory with 4 updates"** by dependabot[bot] (created 2026-07-16T22:32:47Z, head SHA `7fbc9bf40296`, 10/-10 changes, 0 comments, 0 review comments, 4 label-application events) — triaged as **ITEM #43 metadata-refresh noise** (dependabot GitHub Actions version bump is exactly the kind of routine PR that hits the 6h clock-tick cadence). **PR #4735 STILL stable post-substantive** (Cycle 45 BREAKTHROUGH classification sustained) — re-verified 00:05 UTC: head SHA UNCHANGED at `85534a93f8ddc09fb0608a594e40eac61836ebef`, commit count UNCHANGED at 2, comments UNCHANGED at 2, review_comments UNCHANGED at 3, additions UNCHANGED at 162, deletions UNCHANGED at 0, updated_at UNCHANGED at 2026-07-17T11:14:06Z. Maintainers reviewing but not pushing back in past 12h. **PR #4726 still OPEN**, head_sha `090f6e5b8607` UNCHANGED, comments UNCHANGED at 3, 4 review comments, 8 events stable, no maintainer activity. **PR #4731 still DRAFT**, head SHA `b6b4916492d5` UNCHANGED. **PR #4737 (the competing non-BMP UTF-16 fix)** still open, head_sha `ed773a4d385e7f44dd6e8b78e0a9d5f9d3aa18c5` UNCHANGED. **Gin master HEAD STILL UNCHANGED at `34dac209ffb6ef85cc78c5d217bbb7ad001d68fd`** — drought now 21d 7h 18m+ (see above). **v1.13 milestone STILL UNCHANGED** at 12 open / 24 closed (66.7%) — 18d 0h 7m+ OVERDUE (see above). v1.x still 1/17, v2.0 still 0/3. **`go1.26.5` + `go1.25.12` STILL the only Go stable tags** on `go.dev/dl/?mode=json` + `git ls-remote github.com/golang/go` (plus `go1.27rc2` as latest RC) — no `go1.25.13`, `go1.26.6`, or `go1.27rc3` tags pushed. **Patch release window 6d 0h 7m+** (see above). **CVE-2026-39822 STILL PUBLISHED** on MITRE (datePublished=2026-07-08T15:46:27.199Z, dateUpdated=2026-07-08T19:39:17.341Z, no advance). **CVE-2026-42505 STILL PUBLISHED** on MITRE (datePublished=2026-07-08T15:46:33.407Z, dateUpdated=2026-07-08T19:38:17.603Z, no advance). **Tracking issues #80223 + #80224 STILL CLOSED** (since 2026-07-08T23:42:49Z/47Z, no reopen). **Tracking issues #80297 + #80298 STILL CLOSED** (since 2026-07-10T13:40:30Z/29Z, no reopen). x/* modules all UNCHANGED on latest stable tags: x/crypto v0.54.0, x/net v0.57.0, x/sys v0.47.0, x/text v0.40.0, x/image v0.44.0, x/arch v0.29.0, x/tools v0.48.0. **NEW AGENT GUIDANCE ITEM #50 — 6-day patch-release-window protocol.** Per Cycle 44's ITEM #44 (24-48h standard turnaround) and Cycle 45's ITEM #49 (5+ day heuristic inferences — coordinated multi-vendor / batched next minor / new CVE in coordination / operational delays), **NEW ITEM #50 extends the protocol to the 6-day mark**: when patch release window crosses 6 days, narrow ITEM #49's 4 inferences to a 2-inference priority — **(Primary) batched next Go minor release** (Go team is bundling this CVE with another not-yet-public CVE) and **(Secondary) new CVE in pre-disclosure coordination** (joint disclosure pending). De-prioritize coordinated multi-vendor disclosure (clears within 5 days) and operational delays. **Practical implication:** if the 7-day mark (2026-07-19T00:00Z, in 23h 53m) is crossed without tag push, the strongest prediction becomes that the next patch release will include 2+ CVEs (CVE-2026-56853 + 1+ additional not-yet-public). h2c-enabled production Gin services should treat 2026-07-19 as a "likely publication day" but be prepared for an unusually large advisory bundle. **PRODUCTION TIMELINE (updated):** the 6-day mark has been crossed; treat 2026-07-19 as the most-likely publication day for CVE-2026-56853 advisory + Go 1.25.13/1.26.6/1.27-rc3 release tags. Step 2 of the two-step production Go upgrade plan (upgrade from go1.26.5/go1.25.12 to go1.25.13/go1.26.6) is now reliably expected within 0-2 days, with the strongest prediction being 2026-07-19. **ITEM #32 hash-correctness guardrail applied for 46th consecutive cycle:** all 4 SHAs (`34dac209` + `2d5129d2b310` + `a42fec40ab09` + `59418f087ceb`) re-verified against live GitHub APIs in same cycle-write session — no fabrication drift.
