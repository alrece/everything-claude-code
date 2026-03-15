---
paths:
  - "**/*.go"
  - "**/go.mod"
---
# Gin Security

> This file extends [common/security.md](../common/security.md) with Gin framework-specific content.

## Input Validation

Always validate and sanitize user input:

```go
type CreateUserRequest struct {
    Email    string `json:"email" binding:"required,email"`
    Password string `json:"password" binding:"required,min=8"`
    Name     string `json:"name" binding:"required,min=2,max=100"`
}

func (h *UserHandler) Create(c *gin.Context) {
    var req CreateUserRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    // req is now validated
}
```

## Authentication

Implement proper authentication middleware:

```go
func AuthMiddleware(jwtService JWTService) gin.HandlerFunc {
    return func(c *gin.Context) {
        authHeader := c.GetHeader("Authorization")
        if authHeader == "" {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "error": "missing authorization header",
            })
            return
        }

        // Extract Bearer token
        parts := strings.Split(authHeader, " ")
        if len(parts) != 2 || parts[0] != "Bearer" {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "error": "invalid authorization format",
            })
            return
        }

        token := parts[1]
        claims, err := jwtService.Validate(token)
        if err != nil {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "error": "invalid or expired token",
            })
            return
        }

        c.Set("user_id", claims.UserID)
        c.Set("roles", claims.Roles)
        c.Next()
    }
}
```

## CORS Configuration

Configure CORS properly:

```go
// GOOD: Specific origins
config := cors.Config{
    AllowOrigins:     []string{"https://example.com", "https://app.example.com"},
    AllowMethods:     []string{"GET", "POST", "PUT", "DELETE"},
    AllowHeaders:     []string{"Origin", "Content-Type", "Authorization"},
    AllowCredentials: true,
    MaxAge:           12 * time.Hour,
}

// BAD: Wildcard with credentials (security risk)
config := cors.Config{
    AllowOrigins:     []string{"*"}, // Never use with credentials!
    AllowCredentials: true,
}
```

## Rate Limiting

Implement rate limiting on public endpoints:

```go
func RateLimitMiddleware(limiter RateLimiter) gin.HandlerFunc {
    return func(c *gin.Context) {
        key := c.ClientIP() // Or user ID for authenticated requests

        allowed, err := limiter.Allow(key)
        if err != nil {
            c.AbortWithStatusJSON(http.StatusInternalServerError, gin.H{
                "error": "rate limit check failed",
            })
            return
        }

        if !allowed {
            c.AbortWithStatusJSON(http.StatusTooManyRequests, gin.H{
                "error": "too many requests",
                "retry_after": limiter.RetryAfter(key),
            })
            return
        }

        c.Next()
    }
}
```

## SQL Injection Prevention

Never concatenate user input in queries:

```go
// BAD: SQL injection vulnerability
query := fmt.Sprintf("SELECT * FROM users WHERE email = '%s'", email)

// GOOD: Parameterized query (with GORM)
db.Where("email = ?", email).First(&user)

// GOOD: Parameterized query (with native SQL)
db.Raw("SELECT * FROM users WHERE email = ?", email).Scan(&user)
```

## Path Traversal Prevention

Validate file paths:

```go
func DownloadHandler(c *gin.Context) {
    filename := c.Param("filename")

    // Clean the path
    cleanPath := filepath.Clean(filename)

    // Ensure it doesn't contain path traversal
    if strings.Contains(cleanPath, "..") {
        c.AbortWithStatusJSON(http.StatusBadRequest, gin.H{
            "error": "invalid filename",
        })
        return
    }

    // Build full path and verify it's within allowed directory
    fullPath := filepath.Join(uploadsDir, cleanPath)
    if !strings.HasPrefix(fullPath, uploadsDir) {
        c.AbortWithStatusJSON(http.StatusForbidden, gin.H{
            "error": "access denied",
        })
        return
    }

    c.FileAttachment(fullPath, cleanPath)
}
```

## Secrets Management

Never hardcode secrets:

```go
// BAD: Hardcoded secret
var jwtSecret = []byte("my-secret-key")

// GOOD: Load from environment
var jwtSecret = []byte(os.Getenv("JWT_SECRET"))

// GOOD: Validate at startup
func init() {
    if os.Getenv("JWT_SECRET") == "" {
        log.Fatal("JWT_SECRET environment variable is required")
    }
}
```

## Logging Sensitive Data

Never log sensitive information:

```go
// BAD: Logging sensitive data
func (h *UserHandler) Login(c *gin.Context) {
    var req LoginRequest
    c.ShouldBindJSON(&req)
    log.Printf("Login attempt: email=%s, password=%s", req.Email, req.Password)
    // ...
}

// GOOD: Log without sensitive data
func (h *UserHandler) Login(c *gin.Context) {
    var req LoginRequest
    c.ShouldBindJSON(&req)
    log.Printf("Login attempt: email=%s", req.Email)
    // ...
}
```

## HTTPS Enforcement

Enforce HTTPS in production:

```go
func TLSMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        if c.Request.TLS == nil && os.Getenv("ENV") == "production" {
            c.AbortWithStatusJSON(http.StatusForbidden, gin.H{
                "error": "HTTPS required",
            })
            return
        }
        c.Next()
    }
}

// Or use automatic redirect
func RedirectToHTTPS() gin.HandlerFunc {
    return func(c *gin.Context) {
        if c.Request.TLS == nil {
            target := "https://" + c.Request.Host + c.Request.URL.Path
            c.Redirect(http.StatusMovedPermanently, target)
            c.Abort()
            return
        }
        c.Next()
    }
}
```

## Security Headers

Add security headers middleware:

```go
func SecurityHeadersMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Header("X-Content-Type-Options", "nosniff")
        c.Header("X-Frame-Options", "DENY")
        c.Header("X-XSS-Protection", "1; mode=block")
        c.Header("Strict-Transport-Security", "max-age=31536000; includeSubDomains")
        c.Header("Content-Security-Policy", "default-src 'self'")
        c.Next()
    }
}
```

## Request Size Limit

Limit request body size:

```go
func LimitBodySize(maxBytes int64) gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Request.Body = http.MaxBytesReader(c.Writer, c.Request.Body, maxBytes)
        c.Next()
    }
}

// Usage
r.Use(LimitBodySize(10 * 1024 * 1024)) // 10MB limit
```
