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

## Common Mistakes

1. **JWT secret hardcoded** — always use environment variable
2. **Not checking `err` from `ValidateToken`** — expired/invalid tokens cause panics
3. **Storing passwords in plain text** — always use bcrypt
4. **Session cookie not Secure in production** — `Secure: true` when ENV=production
5. **JWT token not validated for expiry** — always check `ExpiresAt`
6. **API key in query string** — headers are more secure (query strings get logged)