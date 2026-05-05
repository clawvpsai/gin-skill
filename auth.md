# Authentication — JWT, Session, Bcrypt, Middleware

## JWT Authentication

```go
package auth

import (
    "net/http"
    "strings"
    "time"
    
    "github.com/gin-gonic/gin"
    "github.com/golang-jwt/jwt/v5"
)

var JWTSecret = []byte(getEnv("JWT_SECRET", "your-secret-key"))

// Claims represent JWT claims
type Claims struct {
    UserID   int    `json:"user_id"`
    Email    string `json:"email"`
    jwt.RegisteredClaims
}

func GenerateToken(userID int, email string) (string, error) {
    claims := Claims{
        UserID: userID,
        Email:  email,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(24 * time.Hour)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
            Issuer:    "myapp",
        },
    }
    
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(JWTSecret)
}

func ValidateToken(tokenString string) (*Claims, error) {
    token, err := jwt.ParseWithClaims(tokenString, &Claims{}, func(token *jwt.Token) (interface{}, error) {
        return JWTSecret, nil
    })
    
    if err != nil {
        return nil, err
    }
    
    if claims, ok := token.Claims.(*Claims); ok && token.Valid {
        return claims, nil
    }
    
    return nil, jwt.ErrSignatureInvalid
}

// JWT Middleware
func JWTAuthMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        authHeader := c.GetHeader("Authorization")
        if authHeader == "" {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "authorization header required"})
            return
        }
        
        // Bearer <token>
        parts := strings.SplitN(authHeader, " ", 2)
        if len(parts) != 2 || parts[0] != "Bearer" {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "invalid authorization format"})
            return
        }
        
        claims, err := ValidateToken(parts[1])
        if err != nil {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "invalid or expired token"})
            return
        }
        
        // Store user info in context
        c.Set("user_id", claims.UserID)
        c.Set("email", claims.Email)
        
        c.Next()
    }
}

// Usage
func ProtectedHandler(c *gin.Context) {
    userID, _ := c.Get("user_id")
    c.JSON(200, gin.H{"user_id": userID})
}
```

## JWT Refresh Token Pattern

```go
// Store both access and refresh tokens
type TokenPair struct {
    AccessToken  string `json:"access_token"`
    RefreshToken string `json:"refresh_token"`
}

func GenerateTokenPair(userID int, email string) (*TokenPair, error) {
    accessClaims := Claims{
        UserID: userID,
        Email:  email,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(15 * time.Minute)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
            Issuer:    "myapp",
            TokenType: "access",
        },
    }
    
    refreshClaims := Claims{
        UserID: userID,
        Email:  email,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(7 * 24 * time.Hour)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
            Issuer:    "myapp",
            TokenType: "refresh",
        },
    }
    
    accessToken := jwt.NewWithClaims(jwt.SigningMethodHS256, accessClaims)
    refreshToken := jwt.NewWithClaims(jwt.SigningMethodHS256, refreshClaims)
    
    accessStr, err := accessToken.SignedString(JWTSecret)
    if err != nil {
        return nil, err
    }
    
    refreshStr, err := refreshToken.SignedString(JWTSecret)
    if err != nil {
        return nil, err
    }
    
    return &TokenPair{AccessToken: accessStr, RefreshToken: refreshStr}, nil
}

func RefreshTokenHandler(c *gin.Context) {
    var req struct {
        RefreshToken string `json:"refresh_token" binding:"required"`
    }
    
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(400, gin.H{"error": "refresh token required"})
        return
    }
    
    claims, err := ValidateToken(req.RefreshToken)
    if err != nil {
        c.JSON(401, gin.H{"error": "invalid or expired refresh token"})
        return
    }
    
    // Verify it's a refresh token
    if claims.TokenType != "refresh" {
        c.JSON(401, gin.H{"error": "not a refresh token"})
        return
    }
    
    // Generate new token pair
    tokens, err := GenerateTokenPair(claims.UserID, claims.Email)
    if err != nil {
        c.JSON(500, gin.H{"error": "failed to generate tokens"})
        return
    }
    
    c.JSON(200, tokens)
}
```

## Password Hashing with Bcrypt

```go
import "golang.org/x/crypto/bcrypt"

func HashPassword(password string) (string, error) {
    bytes, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
    return string(bytes), err
}

func CheckPassword(password, hash string) bool {
    err := bcrypt.CompareHashAndPassword([]byte(hash), []byte(password))
    return err == nil
}

// Usage
func createUser(c *gin.Context) {
    var req struct {
        Email    string `json:"email" binding:"required,email"`
        Password string `json:"password" binding:"required,min=8"`
    }
    
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(400, gin.H{"error": err.Error()})
        return
    }
    
    hashedPassword, err := HashPassword(req.Password)
    if err != nil {
        c.JSON(500, gin.H{"error": "failed to hash password"})
        return
    }
    
    user, err := db.CreateUser(req.Email, hashedPassword)
    if err != nil {
        c.JSON(500, gin.H{"error": "failed to create user"})
        return
    }
    
    c.JSON(201, user)
}

func login(c *gin.Context) {
    var req struct {
        Email    string `json:"email" binding:"required,email"`
        Password string `json:"password" binding:"required"`
    }
    
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(400, gin.H{"error": err.Error()})
        return
    }
    
    user, err := db.GetUserByEmail(req.Email)
    if err != nil {
        c.JSON(401, gin.H{"error": "invalid credentials"})
        return
    }
    
    if !CheckPassword(req.Password, user.PasswordHash) {
        c.JSON(401, gin.H{"error": "invalid credentials"})
        return
    }
    
    token, err := GenerateToken(user.ID, user.Email)
    if err != nil {
        c.JSON(500, gin.H{"error": "failed to generate token"})
        return
    }
    
    c.JSON(200, gin.H{"token": token})
}
```

## Session Authentication

```go
import (
    "github.com/gin-contrib/sessions"
    "github.com/gin-contrib/sessions/cookie"
)

func SetupSession(r *gin.Engine) {
    store := cookie.NewStore([]byte(getEnv("SESSION_SECRET", "secret")))
    store.Options(sessions.Options{
        Path:     "/",
        MaxAge:   86400 * 7, // 7 days
        HttpOnly: true,
        Secure:   getEnv("ENV", "dev") == "production",
        SameSite: http.SameSiteLaxMode,
    })
    r.Use(sessions.Sessions("mysession", store))
}

func loginWithSession(c *gin.Context) {
    var req struct {
        Email    string `json:"email" binding:"required,email"`
        Password string `json:"password" binding:"required"`
    }
    
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(400, gin.H{"error": err.Error()})
        return
    }
    
    user, err := db.GetUserByEmail(req.Email)
    if err != nil || !CheckPassword(req.Password, user.PasswordHash) {
        c.JSON(401, gin.H{"error": "invalid credentials"})
        return
    }
    
    session := sessions.Default(c)
    session.Set("user_id", user.ID)
    session.Set("email", user.Email)
    session.Save()
    
    c.JSON(200, gin.H{"message": "logged in"})
}

func logout(c *gin.Context) {
    session := sessions.Default(c)
    session.Clear()
    session.Save()
    c.JSON(200, gin.H{"message": "logged out"})
}

func requireSession(c *gin.Context) {
    session := sessions.Default(c)
    userID := session.Get("user_id")
    
    if userID == nil {
        c.AbortWithStatusJSON(401, gin.H{"error": "unauthorized"})
        return
    }
    
    c.Set("user_id", userID)
    c.Next()
}
```

## API Key Authentication

```go
func APIKeyAuthMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        apiKey := c.GetHeader("X-API-Key")
        if apiKey == "" {
            apiKey = c.Query("api_key")
        }
        
        if apiKey == "" {
            c.AbortWithStatusJSON(401, gin.H{"error": "api key required"})
            return
        }
        
        valid, userID := validateAPIKey(apiKey)
        if !valid {
            c.AbortWithStatusJSON(401, gin.H{"error": "invalid api key"})
            return
        }
        
        c.Set("user_id", userID)
        c.Next()
    }
}
```

## OAuth2/JWT with Google, GitHub, etc.

```go
// OAuth2 callback handler
func OAuthCallback(c *gin.Context) {
    code := c.Query("code")
    if code == "" {
        c.JSON(400, gin.H{"error": "authorization code required"})
        return
    }
    
    // Exchange code for tokens with OAuth provider
    tokens, err := exchangeCodeForTokens(code)
    if err != nil {
        c.JSON(401, gin.H{"error": "failed to authenticate"})
        return
    }
    
    // Get user info from OAuth provider
    userInfo, err := getOAuthUserInfo(tokens.AccessToken)
    if err != nil {
        c.JSON(500, gin.H{"error": "failed to get user info"})
        return
    }
    
    // Find or create user in database
    user, err := db.FindOrCreateOAuthUser(userInfo)
    if err != nil {
        c.JSON(500, gin.H{"error": "failed to create user"})
        return
    }
    
    // Generate app JWT
    token, err := GenerateToken(user.ID, user.Email)
    if err != nil {
        c.JSON(500, gin.H{"error": "failed to generate token"})
        return
    }
    
    // Redirect to frontend with token or set cookie
    c.SetCookie("token", token, 86400, "/", "", false, true)
    c.Redirect(302, "/dashboard")
}
```

## OAuth2 PKCE (RFC 7636) — Public Clients

For mobile apps and SPAs where a client secret can't be kept confidential, use PKCE (Proof Key for Code Exchange).

**Why PKCE?** In public clients (mobile, SPA), there's no way to keep a client secret confidential — it's always exposed in the app binary or JS bundle. PKCE adds a verifier-challenge step that prevents authorization code interception attacks.

```go
import (
    "crypto/rand"
    "crypto/sha256"
    "encoding/base64"
)

// CodeVerifier is a cryptographically random string (43-128 chars)
func GenerateCodeVerifier() (string, error) {
    b := make([]byte, 32)
    if _, err := rand.Read(b); err != nil {
        return "", err
    }
    // Base64URL encoding without padding
    return base64.RawURLEncoding.EncodeToString(b), nil
}

// CodeChallenge is S256(code_verifier)
func GenerateCodeChallenge(verifier string) string {
    h := sha256.Sum256([]byte(verifier))
    return base64.RawURLEncoding.EncodeToString(h[:])
}

// Authorization URL with PKCE
func BuildAuthURL(state, clientID, redirectURI, codeChallenge string) string {
    v := url.Values{
        "response_type":         {"code"},
        "client_id":             {clientID},
        "redirect_uri":          {redirectURI},
        "scope":                 {"openid profile email"},
        "state":                 {state},
        "code_challenge":        {codeChallenge},
        "code_challenge_method": {"S256"},
    }
    return "https://auth.example.com/authorize?" + v.Encode()
}

// Token exchange with verifier
func ExchangeCodePKCE(ctx context.Context, code, verifier, clientID, redirectURI string) (*TokenResponse, error) {
    params := url.Values{
        "grant_type":    {"authorization_code"},
        "code":          {code},
        "redirect_uri":  {redirectURI},
        "client_id":     {clientID},
        "code_verifier": {verifier}, // Must match what was sent in auth URL
    }
    
    req, _ := http.NewRequestWithContext(ctx, "POST", "https://auth.example.com/token", 
        strings.NewReader(params.Encode()))
    req.Header.Set("Content-Type", "application/x-www-form-urlencoded")
    
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()
    
    var tokens TokenResponse
    if err := json.NewDecoder(resp.Body).Decode(&tokens); err != nil {
        return nil, err
    }
    return &tokens, nil
}

// In Gin handler — initiate OAuth with PKCE
func OAuth2Login(c *gin.Context) {
    verifier, err := GenerateCodeVerifier()
    if err != nil {
        c.JSON(500, gin.H{"error": "failed to generate verifier"})
        return
    }
    challenge := GenerateCodeChallenge(verifier)
    
    state := generateState() // random state parameter
    
    // Store verifier + state in session or short-lived cookie
    c.SetCookie("pkce_verifier", verifier, 300, "/", "", false, true) // 5 min TTL
    c.SetCookie("oauth_state", state, 300, "/", "", false, true)
    
    authURL := BuildAuthURL(state, os.Getenv("OAUTH_CLIENT_ID"), 
        os.Getenv("OAUTH_REDIRECT_URI"), challenge)
    
    c.Redirect(302, authURL)
}

// OAuth2 callback — exchange code with verifier
func OAuth2Callback(c *gin.Context) {
    code := c.Query("code")
    state := c.Query("state")
    storedState, _ := c.Cookie("oauth_state")
    
    if state != storedState {
        c.JSON(400, gin.H{"error": "state mismatch"})
        return
    }
    
    verifier, _ := c.Cookie("pkce_verifier")
    if verifier == "" {
        c.JSON(400, gin.H{"error": "pkce verifier missing"})
        return
    }
    
    tokens, err := ExchangeCodePKCE(c.Request.Context(), code, verifier,
        os.Getenv("OAUTH_CLIENT_ID"), os.Getenv("OAUTH_REDIRECT_URI"))
    if err != nil {
        c.JSON(401, gin.H{"error": "token exchange failed"})
        return
    }
    
    // Use tokens.AccessToken to fetch user info, then create JWT session
    c.JSON(200, gin.H{"access_token": tokens.AccessToken})
}
```

**PKCE Flow Summary:**
1. Client generates random `verifier`, computes `challenge = S256(verifier)`
2. Redirect user to auth server with `code_challenge=challenge&code_challenge_method=S256`
3. Auth server stores `code_challenge`, returns `code`
4. Client calls `/token` with `code` + original `verifier`
5. Server verifies `S256(verifier) == code_challenge` before issuing tokens

## Common Mistakes

1. **JWT secret hardcoded** — always use environment variable
2. **Not checking `err` from `ValidateToken`** — expired/invalid tokens cause panics
3. **Storing passwords in plain text** — always use bcrypt
4. **Session cookie not Secure in production** — `Secure: true` when ENV=production
5. **JWT token not validated for expiry** — always check `ExpiresAt`
6. **API key in query string** — headers are more secure (query strings get logged)
7. **No refresh token rotation** — short-lived access tokens need refresh flow
8. **Not storing TokenType in claims** — can't distinguish access vs refresh tokens
9. **OAuth2 on public clients without PKCE** — code interception vulnerability
10. **Using `jwt/v4`** — deprecated, always use `jwt/v5`

---

## Updated from Research (2026-05)

### OAuth2 PKCE (RFC 7636)
- Required for mobile apps, SPAs, and any public client where client_secret can't be kept confidential
- S256 (SHA-256) code challenge method is required — "plain" method is not secure
- Verifier is stored in short-lived cookie (5 min TTL) during authorization redirect, then sent to token endpoint

### JWT Best Practices
- Always use `jwt/v5` (v4 is deprecated and has known vulnerabilities)
- Short-lived access tokens (15 min) + refresh tokens (7 days) is the standard pattern
- Include `token_type` in claims to distinguish access vs refresh tokens
- Validate `iss` (issuer) and `aud` (audience) in production

### Sources
- RFC 7636 (PKCE): https://www.rfc-editor.org/rfc/rfc7636
- jwt-go v5: https://github.com/golang-jwt/jwt
- OAuth 2.0 Security Best Current Practice: https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics