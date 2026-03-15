---
paths:
  - "**/*.go"
  - "**/go.mod"
---
# Gin 编码风格

> 此文件扩展了 [common/coding-style.md](../common/coding-style.md) 的 Gin 框架特定内容。

## 处理器结构

所有处理器使用 `func(c *gin.Context)` 签名：

```go
// 正确：标准的 Gin 处理器
func (h *UserHandler) GetUser(c *gin.Context) {
    id := c.Param("id")
    user, err := h.service.GetByID(c.Request.Context(), id)
    if err != nil {
        handleError(c, err)
        return
    }
    c.JSON(http.StatusOK, user)
}

// 错误：使用 net/http 签名
func GetUser(w http.ResponseWriter, r *http.Request) {}
```

## 处理器组织

保持处理器专注且在 50 行以内：

```go
// 正确：专注的处理器
func (h *UserHandler) CreateUser(c *gin.Context) {
    var req CreateUserRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    user, err := h.service.Create(c.Request.Context(), req)
    if err != nil {
        handleError(c, err)
        return
    }

    c.JSON(http.StatusCreated, user)
}

// 错误：处理器包含业务逻辑
func (h *UserHandler) CreateUser(c *gin.Context) {
    var req CreateUserRequest
    c.Bind(&req) // 忽略错误！

    // 50+ 行业务逻辑...
    if req.Email != "" {
        // 验证邮箱
    }
    // 更多验证...
    // 数据库操作...
    // 发送邮件...
}
```

## 绑定与验证

始终使用 `ShouldBind*` 而非 `Bind*`：

```go
// 正确：ShouldBind 允许自定义错误处理
var req CreateUserRequest
if err := c.ShouldBindJSON(&req); err != nil {
    c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
    return
}

// 错误：Bind 发送自动响应
var req CreateUserRequest
if err := c.Bind(&req); err != nil {
    return // 自动 400，无法自定义
}
```

## 请求/响应类型

定义类型化的请求和响应结构体：

```go
// 请求类型
type CreateUserRequest struct {
    Email    string `json:"email" binding:"required,email"`
    Password string `json:"password" binding:"required,min=8"`
    Name     string `json:"name" binding:"required"`
}

// 响应类型
type UserResponse struct {
    ID        uint   `json:"id"`
    Email     string `json:"email"`
    Name      string `json:"name"`
    CreatedAt string `json:"createdAt"`
}

// 永远不要直接暴露内部模型
func (h *UserHandler) GetUser(c *gin.Context) {
    user, err := h.service.GetByID(c.Request.Context(), id)
    if err != nil {
        handleError(c, err)
        return
    }

    // 转换为响应类型
    response := UserResponse{
        ID:        user.ID,
        Email:     user.Email,
        Name:      user.Name,
        CreatedAt: user.CreatedAt.Format(time.RFC3339),
    }
    c.JSON(http.StatusOK, response)
}
```

## 路由组织

使用路由组共享中间件：

```go
// 正确：组织良好的路由组
func SetupRoutes(r *gin.Engine, handlers *Handlers) {
    // 公共路由
    public := r.Group("/api/v1")
    {
        public.POST("/login", handlers.Auth.Login)
        public.POST("/register", handlers.Auth.Register)
    }

    // 受保护路由
    protected := r.Group("/api/v1")
    protected.Use(middleware.Auth())
    {
        users := protected.Group("/users")
        {
            users.GET("", handlers.User.List)
            users.GET("/:id", handlers.User.Get)
            users.POST("", handlers.User.Create)
            users.PUT("/:id", handlers.User.Update)
            users.DELETE("/:id", handlers.User.Delete)
        }
    }

    // 管理员路由
    admin := r.Group("/api/v1/admin")
    admin.Use(middleware.Auth(), middleware.AdminOnly())
    {
        admin.GET("/users", handlers.Admin.ListAllUsers)
    }
}

// 错误：扁平路由重复中间件
r.GET("/api/v1/users", middleware.Auth(), handlers.User.List)
r.GET("/api/v1/users/:id", middleware.Auth(), handlers.User.Get)
r.POST("/api/v1/users", middleware.Auth(), handlers.User.Create)
```

## HTTP 状态码

使用适当的状态码：

| 操作 | 状态码 |
|-----------|-------------|
| 创建成功 | 201 Created |
| 更新成功 | 200 OK |
| 删除成功 | 204 No Content |
| 获取成功 | 200 OK |
| 列表成功 | 200 OK |
| 验证错误 | 400 Bad Request |
| 未授权 | 401 Unauthorized |
| 禁止访问 | 403 Forbidden |
| 未找到 | 404 Not Found |
| 冲突 | 409 Conflict |
| 服务器错误 | 500 Internal Server Error |

## 错误处理

一致的错误响应格式：

```go
type ErrorResponse struct {
    Error   string `json:"error"`
    Message string `json:"message,omitempty"`
    Code    string `json:"code,omitempty"`
}

func handleError(c *gin.Context, err error) {
    var appErr *AppError
    if errors.As(err, &appErr) {
        c.JSON(appErr.HTTPStatus, ErrorResponse{
            Error:   appErr.Message,
            Code:    appErr.Code,
        })
        return
    }

    c.JSON(http.StatusInternalServerError, ErrorResponse{
        Error: "Internal server error",
    })
}
```

## 上下文传播

始终使用 `c.Request.Context()` 进行上下文传播：

```go
// 正确：使用请求上下文
func (h *UserHandler) GetUser(c *gin.Context) {
    ctx := c.Request.Context()
    user, err := h.service.GetByID(ctx, id)
    // ...
}

// 错误：使用 context.Background()
func (h *UserHandler) GetUser(c *gin.Context) {
    ctx := context.Background() // 丢失超时/取消
    user, err := h.service.GetByID(ctx, id)
    // ...
}
```
