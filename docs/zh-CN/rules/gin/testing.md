---
paths:
  - "**/*.go"
  - "**/*_test.go"
---
# Gin 测试

> 此文件扩展了 [common/testing.md](../common/testing.md) 的 Gin 框架特定内容。

## 测试框架

使用 `net/http/httptest` 进行 HTTP 测试：

```go
import (
    "net/http"
    "net/http/httptest"
    "testing"

    "github.com/gin-gonic/gin"
    "github.com/stretchr/testify/assert"
)

func init() {
    gin.SetMode(gin.TestMode)
}
```

## 处理器测试

```go
func TestUserHandler_Get(t *testing.T) {
    // 设置
    mockService := &MockUserService{
        GetByIDFunc: func(ctx context.Context, id string) (*User, error) {
            return &User{ID: 1, Name: "测试用户"}, nil
        },
    }
    handler := NewUserHandler(mockService)

    // 创建路由
    router := gin.New()
    router.GET("/users/:id", handler.Get)

    // 创建请求
    req := httptest.NewRequest(http.MethodGet, "/users/1", nil)
    w := httptest.NewRecorder()

    // 执行
    router.ServeHTTP(w, req)

    // 断言
    assert.Equal(t, http.StatusOK, w.Code)

    var response User
    json.Unmarshal(w.Body.Bytes(), &response)
    assert.Equal(t, "测试用户", response.Name)
}
```

## 表驱动测试

```go
func TestUserHandler_Create(t *testing.T) {
    tests := []struct {
        name       string
        payload    interface{}
        setup      func(*MockUserService)
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
            setup: func(m *MockUserService) {
                m.CreateFunc = func(ctx context.Context, req CreateUserRequest) (*User, error) {
                    return &User{ID: 1, Email: req.Email, Name: req.Name}, nil
                }
            },
            wantStatus: http.StatusCreated,
            wantError:  false,
        },
        {
            name: "无效邮箱",
            payload: map[string]string{
                "email":    "invalid-email",
                "password": "password123",
                "name":     "测试用户",
            },
            setup:      func(m *MockUserService) {},
            wantStatus: http.StatusBadRequest,
            wantError:  true,
        },
        {
            name: "缺少必填字段",
            payload: map[string]string{
                "email": "test@example.com",
            },
            setup:      func(m *MockUserService) {},
            wantStatus: http.StatusBadRequest,
            wantError:  true,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            mockService := &MockUserService{}
            tt.setup(mockService)

            handler := NewUserHandler(mockService)
            router := gin.New()
            router.POST("/users", handler.Create)

            body, _ := json.Marshal(tt.payload)
            req := httptest.NewRequest(http.MethodPost, "/users", bytes.NewBuffer(body))
            req.Header.Set("Content-Type", "application/json")

            w := httptest.NewRecorder()
            router.ServeHTTP(w, req)

            assert.Equal(t, tt.wantStatus, w.Code)
        })
    }
}
```

## 中间件测试

```go
func TestAuthMiddleware(t *testing.T) {
    tests := []struct {
        name       string
        token      string
        setup      func(*MockJWTService)
        wantStatus int
        wantNext   bool
    }{
        {
            name:  "有效令牌",
            token: "Bearer valid-token",
            setup: func(m *MockJWTService) {
                m.ValidateFunc = func(token string) (*Claims, error) {
                    return &Claims{UserID: "1"}, nil
                }
            },
            wantStatus: http.StatusOK,
            wantNext:   true,
        },
        {
            name:       "缺少令牌",
            token:      "",
            setup:      func(m *MockJWTService) {},
            wantStatus: http.StatusUnauthorized,
            wantNext:   false,
        },
        {
            name:  "无效令牌",
            token: "Bearer invalid-token",
            setup: func(m *MockJWTService) {
                m.ValidateFunc = func(token string) (*Claims, error) {
                    return nil, errors.New("无效令牌")
                }
            },
            wantStatus: http.StatusUnauthorized,
            wantNext:   false,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            mockJWT := &MockJWTService{}
            tt.setup(mockJWT)

            router := gin.New()
            router.Use(AuthMiddleware(mockJWT))
            router.GET("/protected", func(c *gin.Context) {
                c.Status(http.StatusOK)
            })

            req := httptest.NewRequest(http.MethodGet, "/protected", nil)
            if tt.token != "" {
                req.Header.Set("Authorization", tt.token)
            }

            w := httptest.NewRecorder()
            router.ServeHTTP(w, req)

            assert.Equal(t, tt.wantStatus, w.Code)
        })
    }
}
```

## Mock 服务

```go
type MockUserService struct {
    GetByIDFunc func(ctx context.Context, id string) (*User, error)
    CreateFunc  func(ctx context.Context, req CreateUserRequest) (*User, error)
    ListFunc    func(ctx context.Context, offset, limit int) ([]User, int64, error)
}

func (m *MockUserService) GetByID(ctx context.Context, id string) (*User, error) {
    if m.GetByIDFunc != nil {
        return m.GetByIDFunc(ctx, id)
    }
    return nil, errors.New("未实现")
}

func (m *MockUserService) Create(ctx context.Context, req CreateUserRequest) (*User, error) {
    if m.CreateFunc != nil {
        return m.CreateFunc(ctx, req)
    }
    return nil, errors.New("未实现")
}

func (m *MockUserService) List(ctx context.Context, offset, limit int) ([]User, int64, error) {
    if m.ListFunc != nil {
        return m.ListFunc(ctx, offset, limit)
    }
    return nil, 0, errors.New("未实现")
}
```

## 测试辅助函数

```go
func setupTestRouter() *gin.Engine {
    gin.SetMode(gin.TestMode)
    return gin.New()
}

func makeRequest(router *gin.Engine, method, path string, body interface{}) *httptest.ResponseRecorder {
    var reqBody io.Reader
    if body != nil {
        jsonBody, _ := json.Marshal(body)
        reqBody = bytes.NewBuffer(jsonBody)
    }

    req := httptest.NewRequest(method, path, reqBody)
    req.Header.Set("Content-Type", "application/json")

    w := httptest.NewRecorder()
    router.ServeHTTP(w, req)
    return w
}

func assertJSONResponse(t *testing.T, w *httptest.ResponseRecorder, expectedStatus int, expected interface{}) {
    assert.Equal(t, expectedStatus, w.Code)

    expectedJSON, _ := json.Marshal(expected)
    assert.JSONEq(t, string(expectedJSON), w.Body.String())
}
```

## 覆盖率目标

| 组件 | 目标 |
|-----------|--------|
| 处理器 | 90%+ |
| 中间件 | 95%+ |
| 请求验证 | 100% |
| 错误处理 | 100% |

```bash
# 带覆盖率运行测试
go test -cover ./handler/...

# 生成覆盖率报告
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out
```
