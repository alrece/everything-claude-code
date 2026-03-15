---
paths:
  - "**/*.go"
  - "**/*_test.go"
---
# Gin Testing

> This file extends [common/testing.md](../common/testing.md) with Gin framework-specific content.

## Test Framework

Use `net/http/httptest` for HTTP testing:

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

## Handler Testing

```go
func TestUserHandler_Get(t *testing.T) {
    // Setup
    mockService := &MockUserService{
        GetByIDFunc: func(ctx context.Context, id string) (*User, error) {
            return &User{ID: 1, Name: "Test User"}, nil
        },
    }
    handler := NewUserHandler(mockService)

    // Create router
    router := gin.New()
    router.GET("/users/:id", handler.Get)

    // Create request
    req := httptest.NewRequest(http.MethodGet, "/users/1", nil)
    w := httptest.NewRecorder()

    // Execute
    router.ServeHTTP(w, req)

    // Assert
    assert.Equal(t, http.StatusOK, w.Code)

    var response User
    json.Unmarshal(w.Body.Bytes(), &response)
    assert.Equal(t, "Test User", response.Name)
}
```

## Table-Driven Tests

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
            name: "valid user",
            payload: map[string]string{
                "email":    "test@example.com",
                "password": "password123",
                "name":     "Test User",
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
            name: "invalid email",
            payload: map[string]string{
                "email":    "invalid-email",
                "password": "password123",
                "name":     "Test User",
            },
            setup:      func(m *MockUserService) {},
            wantStatus: http.StatusBadRequest,
            wantError:  true,
        },
        {
            name: "missing required field",
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

## Middleware Testing

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
            name:  "valid token",
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
            name:       "missing token",
            token:      "",
            setup:      func(m *MockJWTService) {},
            wantStatus: http.StatusUnauthorized,
            wantNext:   false,
        },
        {
            name:  "invalid token",
            token: "Bearer invalid-token",
            setup: func(m *MockJWTService) {
                m.ValidateFunc = func(token string) (*Claims, error) {
                    return nil, errors.New("invalid token")
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

## Mock Services

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
    return nil, errors.New("not implemented")
}

func (m *MockUserService) Create(ctx context.Context, req CreateUserRequest) (*User, error) {
    if m.CreateFunc != nil {
        return m.CreateFunc(ctx, req)
    }
    return nil, errors.New("not implemented")
}

func (m *MockUserService) List(ctx context.Context, offset, limit int) ([]User, int64, error) {
    if m.ListFunc != nil {
        return m.ListFunc(ctx, offset, limit)
    }
    return nil, 0, errors.New("not implemented")
}
```

## Test Helpers

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

## Integration Tests

```go
func TestUserAPI_Integration(t *testing.T) {
    if testing.Short() {
        t.Skip("Skipping integration test")
    }

    // Setup test database
    db := setupTestDatabase(t)
    defer db.Close()

    // Setup router with real dependencies
    router := setupTestRouter()
    userRepo := repository.NewUserRepository(db)
    userService := service.NewUserService(userRepo)
    userHandler := handler.NewUserHandler(userService)

    api := router.Group("/api/v1")
    userHandler.RegisterRoutes(api)

    t.Run("create and get user", func(t *testing.T) {
        // Create
        createReq := map[string]string{
            "email":    "test@example.com",
            "password": "password123",
            "name":     "Test User",
        }
        w := makeRequest(router, http.MethodPost, "/api/v1/users", createReq)
        assert.Equal(t, http.StatusCreated, w.Code)

        var createResp User
        json.Unmarshal(w.Body.Bytes(), &createResp)

        // Get
        w = makeRequest(router, http.MethodGet, "/api/v1/users/"+fmt.Sprint(createResp.ID), nil)
        assert.Equal(t, http.StatusOK, w.Code)
    })
}
```

## Coverage Targets

| Component | Target |
|-----------|--------|
| Handlers | 90%+ |
| Middleware | 95%+ |
| Request validation | 100% |
| Error handling | 100% |

```bash
# Run tests with coverage
go test -cover ./handler/...

# Generate coverage report
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out
```
