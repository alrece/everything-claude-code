---
paths:
  - "**/*.go"
  - "**/go.mod"
---
# Gin 模式

> 此文件扩展了 [common/patterns.md](../common/patterns.md) 的 Gin 框架特定内容。

## 带依赖注入的处理器

```go
type UserHandler struct {
    service   UserService
    validator Validator
}

func NewUserHandler(service UserService, validator Validator) *UserHandler {
    return &UserHandler{
        service:   service,
        validator: validator,
    }
}

func (h *UserHandler) RegisterRoutes(r *gin.RouterGroup) {
    users := r.Group("/users")
    {
        users.GET("", h.List)
        users.GET("/:id", h.Get)
        users.POST("", h.Create)
        users.PUT("/:id", h.Update)
        users.DELETE("/:id", h.Delete)
    }
}
```

## 中间件模式

```go
// 认证中间件
func AuthMiddleware(authService AuthService) gin.HandlerFunc {
    return func(c *gin.Context) {
        token := c.GetHeader("Authorization")
        if token == "" {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "error": "缺少授权令牌",
            })
            return
        }

        user, err := authService.ValidateToken(token)
        if err != nil {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "error": "无效令牌",
            })
            return
        }

        c.Set("user", user)
        c.Next()
    }
}

// 基于角色的中间件
func RequireRole(roles ...string) gin.HandlerFunc {
    return func(c *gin.Context) {
        user, exists := c.Get("user")
        if !exists {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "未授权"})
            return
        }

        u := user.(*User)
        for _, role := range roles {
            if u.Role == role {
                c.Next()
                return
            }
        }

        c.AbortWithStatusJSON(http.StatusForbidden, gin.H{"error": "权限不足"})
    }
}
```

## 请求验证模式

```go
// 使用结构体标签进行验证
type CreateUserRequest struct {
    Email    string `json:"email" binding:"required,email"`
    Password string `json:"password" binding:"required,min=8,max=72"`
    Name     string `json:"name" binding:"required,min=2,max=100"`
    Phone    string `json:"phone" binding:"omitempty,e164"`
    Age      int    `json:"age" binding:"omitempty,min=0,max=150"`
}

// 自定义验证器
func RegisterCustomValidators(v *validator.Validate) {
    v.RegisterValidation("e164", func(fl validator.FieldLevel) bool {
        return regexp.MustCompile(`^\+[1-9]\d{1,14}$`).MatchString(fl.Field().String())
    })
}
```

## 分页模式

```go
type PaginationRequest struct {
    Page     int `form:"page" binding:"omitempty,min=1"`
    PageSize int `form:"pageSize" binding:"omitempty,min=1,max=100"`
}

func (p *PaginationRequest) GetOffset() int {
    if p.Page <= 0 {
        p.Page = 1
    }
    if p.PageSize <= 0 {
        p.PageSize = 10
    }
    return (p.Page - 1) * p.PageSize
}

func (p *PaginationRequest) GetLimit() int {
    if p.PageSize <= 0 {
        return 10
    }
    return p.PageSize
}

type PaginatedResponse struct {
    Items      interface{} `json:"items"`
    Total      int64       `json:"total"`
    Page       int         `json:"page"`
    PageSize   int         `json:"pageSize"`
    TotalPages int         `json:"totalPages"`
}

func (h *UserHandler) List(c *gin.Context) {
    var req PaginationRequest
    if err := c.ShouldBindQuery(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    items, total, err := h.service.List(c.Request.Context(), req.GetOffset(), req.GetLimit())
    if err != nil {
        handleError(c, err)
        return
    }

    c.JSON(http.StatusOK, PaginatedResponse{
        Items:      items,
        Total:      total,
        Page:       req.Page,
        PageSize:   req.PageSize,
        TotalPages: int(math.Ceil(float64(total) / float64(req.PageSize))),
    })
}
```

## 错误类型模式

```go
type AppError struct {
    Code       string
    Message    string
    HTTPStatus int
    Err        error
}

func (e *AppError) Error() string {
    if e.Err != nil {
        return fmt.Sprintf("%s: %v", e.Message, e.Err)
    }
    return e.Message
}

func (e *AppError) Unwrap() error {
    return e.Err
}

// 预定义错误
var (
    ErrNotFound = &AppError{
        Code:       "NOT_FOUND",
        Message:    "资源未找到",
        HTTPStatus: http.StatusNotFound,
    }
    ErrUnauthorized = &AppError{
        Code:       "UNAUTHORIZED",
        Message:    "未授权访问",
        HTTPStatus: http.StatusUnauthorized,
    }
    ErrValidation = &AppError{
        Code:       "VALIDATION_ERROR",
        Message:    "验证失败",
        HTTPStatus: http.StatusBadRequest,
    }
)

// 包装错误带上下文
func WrapError(err error, appErr *AppError) *AppError {
    return &AppError{
        Code:       appErr.Code,
        Message:    appErr.Message,
        HTTPStatus: appErr.HTTPStatus,
        Err:        err,
    }
}
```

## 请求 ID 模式

```go
func RequestIDMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        requestID := c.GetHeader("X-Request-ID")
        if requestID == "" {
            requestID = uuid.New().String()
        }

        c.Set("request_id", requestID)
        c.Header("X-Request-ID", requestID)

        c.Next()
    }
}

// 在处理器中使用
func (h *UserHandler) Get(c *gin.Context) {
    requestID := c.GetString("request_id")
    h.logger.Info("处理请求", "request_id", requestID, "path", c.Request.URL.Path)
    // ...
}
```

## 速率限制模式

```go
func RateLimitMiddleware(limiter *rate.Limiter) gin.HandlerFunc {
    return func(c *gin.Context) {
        if !limiter.Allow() {
            c.AbortWithStatusJSON(http.StatusTooManyRequests, gin.H{
                "error": "超过速率限制",
            })
            return
        }
        c.Next()
    }
}

// 按客户端 IP 限制
func RateLimitByIP(rps int) gin.HandlerFunc {
    limiters := sync.Map{}

    return func(c *gin.Context) {
        ip := c.ClientIP()

        limiter, _ := limiters.LoadOrStore(ip, rate.NewLimiter(rate.Limit(rps), rps*2))

        if !limiter.(*rate.Limiter).Allow() {
            c.AbortWithStatusJSON(http.StatusTooManyRequests, gin.H{
                "error": "超过速率限制",
            })
            return
        }

        c.Next()
    }
}
```

## CORS 模式

```go
func CORSMiddleware(config CORSConfig) gin.HandlerFunc {
    return cors.New(cors.Config{
        AllowOrigins:     config.AllowedOrigins,
        AllowMethods:     []string{"GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"},
        AllowHeaders:     []string{"Origin", "Content-Type", "Authorization", "X-Request-ID"},
        ExposeHeaders:    []string{"X-Request-ID"},
        AllowCredentials: config.AllowCredentials,
        MaxAge:           12 * time.Hour,
    })
}
```

## 参考

详细的 Go 模式请参阅 skill: `golang-patterns`。
高级 Gin 模式请参阅 skill: `gin-best-practices`。
