---
paths:
  - "**/*.go"
  - "**/go.mod"
---
# Gin Coding Style

> This file extends [common/coding-style.md](../common/coding-style.md) with Gin framework-specific content.

## Handler Structure

Use `func(c *gin.Context)` signature for all handlers:

```go
// GOOD: Proper Gin handler
func (h *UserHandler) GetUser(c *gin.Context) {
    id := c.Param("id")
    user, err := h.service.GetByID(c.Request.Context(), id)
    if err != nil {
        handleError(c, err)
        return
    }
    c.JSON(http.StatusOK, user)
}

// BAD: Using net/http signature
func GetUser(w http.ResponseWriter, r *http.Request) {}
```

## Handler Organization

Keep handlers focused and under 50 lines:

```go
// GOOD: Focused handler
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

// BAD: Handler with business logic
func (h *UserHandler) CreateUser(c *gin.Context) {
    var req CreateUserRequest
    c.Bind(&req) // Ignoring error!

    // 50+ lines of business logic...
    if req.Email != "" {
        // validate email
    }
    // more validation...
    // database operations...
    // email sending...
}
```

## Binding and Validation

Always use `ShouldBind*` over `Bind*`:

```go
// GOOD: ShouldBind allows custom error handling
var req CreateUserRequest
if err := c.ShouldBindJSON(&req); err != nil {
    c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
    return
}

// BAD: Bind sends automatic response
var req CreateUserRequest
if err := c.Bind(&req); err != nil {
    return // Automatic 400, can't customize
}
```

## Request/Response Types

Define typed request and response structs:

```go
// Request types
type CreateUserRequest struct {
    Email    string `json:"email" binding:"required,email"`
    Password string `json:"password" binding:"required,min=8"`
    Name     string `json:"name" binding:"required"`
}

// Response types
type UserResponse struct {
    ID        uint   `json:"id"`
    Email     string `json:"email"`
    Name      string `json:"name"`
    CreatedAt string `json:"createdAt"`
}

// Never expose internal models directly
func (h *UserHandler) GetUser(c *gin.Context) {
    user, err := h.service.GetByID(c.Request.Context(), id)
    if err != nil {
        handleError(c, err)
        return
    }

    // Transform to response type
    response := UserResponse{
        ID:        user.ID,
        Email:     user.Email,
        Name:      user.Name,
        CreatedAt: user.CreatedAt.Format(time.RFC3339),
    }
    c.JSON(http.StatusOK, response)
}
```

## Route Organization

Use route groups for shared middleware:

```go
// GOOD: Organized route groups
func SetupRoutes(r *gin.Engine, handlers *Handlers) {
    // Public routes
    public := r.Group("/api/v1")
    {
        public.POST("/login", handlers.Auth.Login)
        public.POST("/register", handlers.Auth.Register)
    }

    // Protected routes
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

    // Admin routes
    admin := r.Group("/api/v1/admin")
    admin.Use(middleware.Auth(), middleware.AdminOnly())
    {
        admin.GET("/users", handlers.Admin.ListAllUsers)
    }
}

// BAD: Flat routes with repeated middleware
r.GET("/api/v1/users", middleware.Auth(), handlers.User.List)
r.GET("/api/v1/users/:id", middleware.Auth(), handlers.User.Get)
r.POST("/api/v1/users", middleware.Auth(), handlers.User.Create)
```

## HTTP Status Codes

Use appropriate status codes:

| Operation | Status Code |
|-----------|-------------|
| Create success | 201 Created |
| Update success | 200 OK |
| Delete success | 204 No Content |
| Get success | 200 OK |
| List success | 200 OK |
| Validation error | 400 Bad Request |
| Unauthorized | 401 Unauthorized |
| Forbidden | 403 Forbidden |
| Not found | 404 Not Found |
| Conflict | 409 Conflict |
| Server error | 500 Internal Server Error |

## Error Handling

Consistent error response format:

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

## Context Propagation

Always use `c.Request.Context()` for context propagation:

```go
// GOOD: Using request context
func (h *UserHandler) GetUser(c *gin.Context) {
    ctx := c.Request.Context()
    user, err := h.service.GetByID(ctx, id)
    // ...
}

// BAD: Using context.Background()
func (h *UserHandler) GetUser(c *gin.Context) {
    ctx := context.Background() // Lost timeout/cancellation
    user, err := h.service.GetByID(ctx, id)
    // ...
}
```
