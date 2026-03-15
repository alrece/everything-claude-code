---
paths:
  - "**/*.go"
  - "**/go.mod"
---
# Gin 安全

> 此文件扩展了 [common/security.md](../common/security.md) 的 Gin 框架特定内容。

## 输入验证

始终验证和清理用户输入：

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
    // req 现在已验证
}
```

## 认证

实现正确的认证中间件：

```go
func AuthMiddleware(jwtService JWTService) gin.HandlerFunc {
    return func(c *gin.Context) {
        authHeader := c.GetHeader("Authorization")
        if authHeader == "" {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "error": "缺少授权头",
            })
            return
        }

        // 提取 Bearer 令牌
        parts := strings.Split(authHeader, " ")
        if len(parts) != 2 || parts[0] != "Bearer" {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "error": "无效的授权格式",
            })
            return
        }

        token := parts[1]
        claims, err := jwtService.Validate(token)
        if err != nil {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "error": "无效或过期的令牌",
            })
            return
        }

        c.Set("user_id", claims.UserID)
        c.Set("roles", claims.Roles)
        c.Next()
    }
}
```

## CORS 配置

正确配置 CORS：

```go
// 正确：指定来源
config := cors.Config{
    AllowOrigins:     []string{"https://example.com", "https://app.example.com"},
    AllowMethods:     []string{"GET", "POST", "PUT", "DELETE"},
    AllowHeaders:     []string{"Origin", "Content-Type", "Authorization"},
    AllowCredentials: true,
    MaxAge:           12 * time.Hour,
}

// 错误：通配符带凭据（安全风险）
config := cors.Config{
    AllowOrigins:     []string{"*"}, // 永远不要与凭据一起使用！
    AllowCredentials: true,
}
```

## 速率限制

在公共端点实现速率限制：

```go
func RateLimitMiddleware(limiter RateLimiter) gin.HandlerFunc {
    return func(c *gin.Context) {
        key := c.ClientIP() // 或认证请求使用用户 ID

        allowed, err := limiter.Allow(key)
        if err != nil {
            c.AbortWithStatusJSON(http.StatusInternalServerError, gin.H{
                "error": "速率限制检查失败",
            })
            return
        }

        if !allowed {
            c.AbortWithStatusJSON(http.StatusTooManyRequests, gin.H{
                "error": "请求过多",
                "retry_after": limiter.RetryAfter(key),
            })
            return
        }

        c.Next()
    }
}
```

## SQL 注入防护

永远不要在查询中拼接用户输入：

```go
// 错误：SQL 注入漏洞
query := fmt.Sprintf("SELECT * FROM users WHERE email = '%s'", email)

// 正确：参数化查询（使用 GORM）
db.Where("email = ?", email).First(&user)

// 正确：参数化查询（原生 SQL）
db.Raw("SELECT * FROM users WHERE email = ?", email).Scan(&user)
```

## 路径遍历防护

验证文件路径：

```go
func DownloadHandler(c *gin.Context) {
    filename := c.Param("filename")

    // 清理路径
    cleanPath := filepath.Clean(filename)

    // 确保不包含路径遍历
    if strings.Contains(cleanPath, "..") {
        c.AbortWithStatusJSON(http.StatusBadRequest, gin.H{
            "error": "无效的文件名",
        })
        return
    }

    // 构建完整路径并验证在允许目录内
    fullPath := filepath.Join(uploadsDir, cleanPath)
    if !strings.HasPrefix(fullPath, uploadsDir) {
        c.AbortWithStatusJSON(http.StatusForbidden, gin.H{
            "error": "访问被拒绝",
        })
        return
    }

    c.FileAttachment(fullPath, cleanPath)
}
```

## 密钥管理

永远不要硬编码密钥：

```go
// 错误：硬编码密钥
var jwtSecret = []byte("my-secret-key")

// 正确：从环境变量加载
var jwtSecret = []byte(os.Getenv("JWT_SECRET"))

// 正确：启动时验证
func init() {
    if os.Getenv("JWT_SECRET") == "" {
        log.Fatal("需要 JWT_SECRET 环境变量")
    }
}
```

## 记录敏感数据

永远不要记录敏感信息：

```go
// 错误：记录敏感数据
func (h *UserHandler) Login(c *gin.Context) {
    var req LoginRequest
    c.ShouldBindJSON(&req)
    log.Printf("登录尝试: email=%s, password=%s", req.Email, req.Password)
    // ...
}

// 正确：不记录敏感数据
func (h *UserHandler) Login(c *gin.Context) {
    var req LoginRequest
    c.ShouldBindJSON(&req)
    log.Printf("登录尝试: email=%s", req.Email)
    // ...
}
```

## HTTPS 强制

在生产环境强制 HTTPS：

```go
func TLSMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        if c.Request.TLS == nil && os.Getenv("ENV") == "production" {
            c.AbortWithStatusJSON(http.StatusForbidden, gin.H{
                "error": "需要 HTTPS",
            })
            return
        }
        c.Next()
    }
}

// 或使用自动重定向
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

## 安全头

添加安全头中间件：

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

## 请求大小限制

限制请求体大小：

```go
func LimitBodySize(maxBytes int64) gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Request.Body = http.MaxBytesReader(c.Writer, c.Request.Body, maxBytes)
        c.Next()
    }
}

// 使用
r.Use(LimitBodySize(10 * 1024 * 1024)) // 10MB 限制
```
