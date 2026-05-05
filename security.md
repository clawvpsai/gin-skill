# Security — Headers, CORS, Rate Limiting, Hardening

## Security Headers

```go
func SecurityHeaders() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Header("X-Content-Type-Options", "nosniff")
        c.Header("X-Frame-Options", "DENY")
        c.Header("X-XSS-Protection", "1; mode=block")
        c.Header("Referrer-Policy", "strict-origin-when-cross-origin")
        c.Header("Content-Security-Policy", "default-src 'self'")
        c.Header("Permissions-Policy", "geolocation=(), microphone=(), camera=()")
        c.Header("Strict-Transport-Security", "max-age=31536000; includeSubDomains")
        c.Next()
    }
}

r.Use(SecurityHeaders())
```

**Note:** `X-XSS-Protection` is deprecated in modern browsers; Content-Security-Policy is the proper replacement. Include it for legacy browser compatibility.

## Content-Security-Policy with Reporting

```go
func CSPWithReporting() gin.HandlerFunc {
    return func(c *gin.Context) {
        // Report violations to /csp-report endpoint
        c.Header("Content-Security-Policy", 
            "default-src 'self'; report-uri /csp-report; style-src 'self' 'unsafe-inline'")
        c.Next()
    }
}

// CSP violation report handler
func cspReportHandler(c *gin.Context) {
    var report struct {
        CSPReport struct {
            DocumentURI  string `json:"document-uri"`
            ViolatedDirective string `json:"violated-directive"`
            OriginalPolicy string `json:"original-policy"`
        } `json:"csp-report"`
    }
    
    if err := c.ShouldBindJSON(&report); err != nil {
        c.JSON(400, gin.H{"error": "invalid report"})
        return
    }
    
    // Log CSP violations for review
    log.Printf("CSP violation: %s — %s", 
        report.CSPReport.DocumentURI, 
        report.CSPReport.ViolatedDirective)
    
    c.Status(204)
}
```

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
    "content": template.HTML(user.Content), // only after sanitization
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

---

## Updated from Research (2026-05)

### New Security Headers
- **Strict-Transport-Security (HSTS)** — enforces HTTPS for all communications, prevents SSL stripping attacks. Add `; includeSubDomains` to cover subdomains.
- **Permissions-Policy** — replaces older `X-Frame-Options` approach for controlling browser features
- **security.txt (RFC 9116)** — standardized disclosure contact file at `/.well-known/security.txt`

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