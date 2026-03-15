---
description: Enforce TDD workflow for Gin handlers. Write httptest tests first, then implement. Verify handlers with comprehensive test coverage.
---

# Gin TDD Command

This command enforces test-driven development methodology for Gin framework handlers and middleware.

## What This Command Does

1. **Define Handler Interface**: Scaffold handler signatures first
2. **Write httptest Tests**: Create comprehensive test cases (RED)
3. **Run Tests**: Verify tests fail for the right reason
4. **Implement Handler**: Write minimal code to pass (GREEN)
5. **Refactor**: Improve while keeping tests green
6. **Check Coverage**: Ensure 80%+ handler coverage

## When to Use

Use `/gin-test` when:
- Implementing new Gin handlers
- Adding middleware with tests
- Testing API endpoints
- Fixing handler bugs (write failing test first)
- Building critical API logic

## TDD Cycle

```
RED     → Write failing httptest for handler
GREEN   → Implement minimal handler to pass
REFACTOR → Improve handler, tests stay green
REPEAT  → Next test case
```

## Example Session

````
User: /gin-test I need a handler to create a user

Agent:
# TDD Session: CreateUser Handler

## Step 1: Define Handler Interface

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

// CreateUser handles POST /users
func (h *UserHandler) CreateUser(c *gin.Context) {
    // TODO: Implement
    c.Status(http.StatusNotImplemented)
}
```

## Step 2: Write httptest Tests (RED)

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
            name: "valid user",
            payload: map[string]string{
                "email":    "test@example.com",
                "password": "password123",
                "name":     "Test User",
            },
            wantStatus: http.StatusCreated,
            wantError:  false,
        },
        {
            name: "missing email",
            payload: map[string]string{
                "password": "password123",
                "name":     "Test User",
            },
            wantStatus: http.StatusBadRequest,
            wantError:  true,
        },
        {
            name: "invalid email",
            payload: map[string]string{
                "email":    "invalid-email",
                "password": "password123",
                "name":     "Test User",
            },
            wantStatus: http.StatusBadRequest,
            wantError:  true,
        },
        {
            name: "short password",
            payload: map[string]string{
                "email":    "test@example.com",
                "password": "short",
                "name":     "Test User",
            },
            wantStatus: http.StatusBadRequest,
            wantError:  true,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // Setup router
            router := gin.New()
            mockService := &MockUserService{}
            handler := NewUserHandler(mockService)
            router.POST("/users", handler.CreateUser)

            // Create request
            body, _ := json.Marshal(tt.payload)
            req := httptest.NewRequest(http.MethodPost, "/users", bytes.NewBuffer(body))
            req.Header.Set("Content-Type", "application/json")

            // Execute
            w := httptest.NewRecorder()
            router.ServeHTTP(w, req)

            // Assert
            assert.Equal(t, tt.wantStatus, w.Code)
        })
    }
}
```

## Step 3: Run Tests - Verify FAIL

```bash
$ go test ./handler/...

--- FAIL: TestCreateUser (0.00s)
    --- FAIL: TestCreateUser/valid_user (0.00s)
        Error: Not equal: 201 != 501
```

✓ Tests fail as expected (not implemented).

## Step 4: Implement Handler (GREEN)

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
        c.JSON(http.StatusInternalServerError, gin.H{"error": "failed to create user"})
        return
    }

    c.JSON(http.StatusCreated, CreateUserResponse{
        ID:    user.ID,
        Email: user.Email,
        Name:  user.Name,
    })
}
```

## Step 5: Run Tests - Verify PASS

```bash
$ go test ./handler/...

PASS
ok      project/internal/handler   0.015s
```

✓ All tests passing!

## Step 6: Check Coverage

```bash
$ go test -cover ./handler/...

PASS
coverage: 92.5% of statements
ok      project/internal/handler   0.015s
```

✓ Coverage: 92.5%

## TDD Complete!
````

## Test Patterns

### httptest Setup
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

### Table-Driven Handler Tests
```go
tests := []struct {
    name       string
    payload    interface{}
    headers    map[string]string
    wantStatus int
    wantBody   string
}{
    {"success", validPayload, nil, 201, ""},
    {"unauthorized", validPayload, noAuth, 401, ""},
    {"bad request", invalidPayload, nil, 400, "error"},
}
```

### Mock Services
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

### Middleware Testing
```go
func TestAuthMiddleware(t *testing.T) {
    gin.SetMode(gin.TestMode)

    tests := []struct {
        name       string
        token      string
        wantStatus int
    }{
        {"valid token", "valid-token", 200},
        {"missing token", "", 401},
        {"invalid token", "invalid-token", 401},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            router := gin.New()
            router.Use(AuthMiddleware(mockAuth))
            router.GET("/protected", func(c *gin.Context) {
                c.Status(200)
            })

            req := httptest.NewRequest(http.MethodGet, "/protected", nil)
            if tt.token != "" {
                req.Header.Set("Authorization", "Bearer "+tt.token)
            }

            w := httptest.NewRecorder()
            router.ServeHTTP(w, req)

            assert.Equal(t, tt.wantStatus, w.Code)
        })
    }
}
```

## Coverage Commands

```bash
# Basic coverage
go test -cover ./handler/...

# Coverage profile
go test -coverprofile=coverage.out ./handler/...

# View in browser
go tool cover -html=coverage.out

# With race detection
go test -race -cover ./handler/...
```

## Coverage Targets

| Handler Type | Target |
|--------------|--------|
| Critical endpoints (auth, payment) | 100% |
| Public APIs | 90%+ |
| General handlers | 80%+ |
| Middleware | 90%+ |

## TDD Best Practices

**DO:**
- Write test FIRST, before any implementation
- Use `httptest` for HTTP testing
- Mock external dependencies (services, databases)
- Test all HTTP status codes
- Include edge cases (empty body, invalid JSON)

**DON'T:**
- Skip the RED phase
- Test implementation details
- Use real databases in unit tests
- Ignore error paths
- Skip authentication tests

## Related Commands

- `/gin-build` - Fix build errors
- `/gin-review` - Review code after implementation
- `/go-test` - General Go testing

## Related

- Skill: `skills/golang-testing/`
- Agent: `agents/e2e-runner.md`
