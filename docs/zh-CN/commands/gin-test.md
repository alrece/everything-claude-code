---
description: Gin 处理器的 TDD 工作流。先写 httptest 测试，再实现。确保处理器有全面的测试覆盖。
---

# Gin TDD 命令

此命令为 Gin 框架处理器和中间件强制执行测试驱动开发方法论。

## 命令功能

1. **定义处理器接口**：首先搭建处理器签名
2. **编写 httptest 测试**：创建全面的测试用例（RED）
3. **运行测试**：验证测试因正确原因失败
4. **实现处理器**：编写最小代码通过测试（GREEN）
5. **重构**：改进代码，保持测试通过
6. **检查覆盖率**：确保 80%+ 处理器覆盖

## 使用场景

在以下情况使用 `/gin-test`：
* 实现新的 Gin 处理器
* 添加带测试的中间件
* 测试 API 端点
* 修复处理器 bug（先写失败测试）
* 构建关键 API 逻辑

## TDD 循环

```
RED     → 为处理器编写失败的 httptest
GREEN   → 实现最小处理器代码通过测试
REFACTOR → 改进处理器，测试保持通过
REPEAT  → 下一个测试用例
```

## 示例会话

````
用户: /gin-test 我需要一个创建用户的处理器

代理:
# TDD 会话: CreateUser 处理器

## 步骤 1: 定义处理器接口

```go
// handler/user.go
package handler

import "github.com/gin-gonic/gin"

type UserHandler struct {
    service UserService
}

func NewUserHandler(service UserService) *UserHandler {
    return &UserHandler{service: service}
}

// CreateUser 处理 POST /users
func (h *UserHandler) CreateUser(c *gin.Context) {
    // TODO: 实现
    c.Status(http.StatusNotImplemented)
}
```

## 步骤 2: 编写 httptest 测试 (RED)

```go
// handler/user_test.go
package handler

import (
    "bytes"
    "encoding/json"
    "net/http"
    "net/http/httptest"
    "testing"

    "github.com/gin-gonic/gin"
    "github.com/stretchr/testify/assert"
)

func TestCreateUser(t *testing.T) {
    gin.SetMode(gin.TestMode)

    tests := []struct {
        name       string
        payload    interface{}
        wantStatus int
        wantError  bool
    }{
        {
            name: "有效用户",
            payload: map[string]string{
                "email":    "test@example.com",
                "password": "password123",
                "name":     "测试用户",
            },
            wantStatus: http.StatusCreated,
            wantError:  false,
        },
        {
            name: "缺少邮箱",
            payload: map[string]string{
                "password": "password123",
                "name":     "测试用户",
            },
            wantStatus: http.StatusBadRequest,
            wantError:  true,
        },
        {
            name: "无效邮箱",
            payload: map[string]string{
                "email":    "invalid-email",
                "password": "password123",
                "name":     "测试用户",
            },
            wantStatus: http.StatusBadRequest,
            wantError:  true,
        },
        {
            name: "密码太短",
            payload: map[string]string{
                "email":    "test@example.com",
                "password": "short",
                "name":     "测试用户",
            },
            wantStatus: http.StatusBadRequest,
            wantError:  true,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // 设置路由
            router := gin.New()
            mockService := &MockUserService{}
            handler := NewUserHandler(mockService)
            router.POST("/users", handler.CreateUser)

            // 创建请求
            body, _ := json.Marshal(tt.payload)
            req := httptest.NewRequest(http.MethodPost, "/users", bytes.NewBuffer(body))
            req.Header.Set("Content-Type", "application/json")

            // 执行
            w := httptest.NewRecorder()
            router.ServeHTTP(w, req)

            // 断言
            assert.Equal(t, tt.wantStatus, w.Code)
        })
    }
}
```

## 步骤 3: 运行测试 - 验证失败

```bash
$ go test ./handler/...

--- FAIL: TestCreateUser (0.00s)
    --- FAIL: TestCreateUser/有效用户 (0.00s)
        Error: Not equal: 201 != 501
```

✓ 测试按预期失败（未实现）。

## 步骤 4: 实现处理器 (GREEN)

```go
// handler/user.go
package handler

import (
    "net/http"

    "github.com/gin-gonic/gin"
)

type CreateUserRequest struct {
    Email    string `json:"email" binding:"required,email"`
    Password string `json:"password" binding:"required,min=8"`
    Name     string `json:"name" binding:"required"`
}

type CreateUserResponse struct {
    ID    uint   `json:"id"`
    Email string `json:"email"`
    Name  string `json:"name"`
}

func (h *UserHandler) CreateUser(c *gin.Context) {
    var req CreateUserRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    user, err := h.service.CreateUser(c.Request.Context(), req.Email, req.Password, req.Name)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "创建用户失败"})
        return
    }

    c.JSON(http.StatusCreated, CreateUserResponse{
        ID:    user.ID,
        Email: user.Email,
        Name:  user.Name,
    })
}
```

## 步骤 5: 运行测试 - 验证通过

```bash
$ go test ./handler/...

PASS
ok      project/internal/handler   0.015s
```

✓ 所有测试通过！

## 步骤 6: 检查覆盖率

```bash
$ go test -cover ./handler/...

PASS
coverage: 92.5% of statements
ok      project/internal/handler   0.015s
```

✓ 覆盖率: 92.5%

## TDD 完成！
````

## 测试模式

### httptest 设置
```go
func setupTestRouter() *gin.Engine {
    gin.SetMode(gin.TestMode)
    router := gin.New()
    return router
}

func makeRequest(router *gin.Engine, method, path string, body interface{}) *httptest.ResponseRecorder {
    jsonBody, _ := json.Marshal(body)
    req := httptest.NewRequest(method, path, bytes.NewBuffer(jsonBody))
    req.Header.Set("Content-Type", "application/json")
    w := httptest.NewRecorder()
    router.ServeHTTP(w, req)
    return w
}
```

### 表驱动处理器测试
```go
tests := []struct {
    name       string
    payload    interface{}
    headers    map[string]string
    wantStatus int
    wantBody   string
}{
    {"成功", validPayload, nil, 201, ""},
    {"未授权", validPayload, noAuth, 401, ""},
    {"错误请求", invalidPayload, nil, 400, "error"},
}
```

### Mock 服务
```go
type MockUserService struct {
    CreateUserFunc func(ctx context.Context, email, password, name string) (*User, error)
}

func (m *MockUserService) CreateUser(ctx context.Context, email, password, name string) (*User, error) {
    if m.CreateUserFunc != nil {
        return m.CreateUserFunc(ctx, email, password, name)
    }
    return &User{ID: 1, Email: email, Name: name}, nil
}
```

## 覆盖率命令

```bash
# 基本覆盖率
go test -cover ./handler/...

# 覆盖率配置文件
go test -coverprofile=coverage.out ./handler/...

# 在浏览器中查看
go tool cover -html=coverage.out

# 带竞态检测
go test -race -cover ./handler/...
```

## 覆盖率目标

| 处理器类型 | 目标 |
|--------------|--------|
| 关键端点（认证、支付） | 100% |
| 公共 API | 90%+ |
| 通用处理器 | 80%+ |
| 中间件 | 90%+ |

## TDD 最佳实践

**应该做：**
* 在任何实现之前先写测试
* 使用 `httptest` 进行 HTTP 测试
* Mock 外部依赖（服务、数据库）
* 测试所有 HTTP 状态码
* 包含边界情况（空请求体、无效 JSON）

**不应该做：**
* 跳过 RED 阶段
* 测试实现细节
* 在单元测试中使用真实数据库
* 忽略错误路径
* 跳过认证测试

## 相关命令

* `/gin-build` - 修复构建错误
* `/gin-review` - 实现后审查代码
* `/go-test` - 通用 Go 测试

## 相关

* 技能: `skills/golang-testing/`
* 代理: `agents/e2e-runner.md`
