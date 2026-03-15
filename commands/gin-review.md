---
description: Comprehensive Gin framework code review for RESTful API design, middleware patterns, routing conventions, and security. Invokes the gin-reviewer agent.
---

# Gin Code Review

This command invokes the **gin-reviewer** agent for comprehensive Gin-specific code review.

## What This Command Does

1. **Identify Gin Changes**: Find modified handler/route files via `git diff`
2. **Run Static Analysis**: Execute `go vet`, `staticcheck`, and linters
3. **Security Scan**: Check for SQL injection, CORS, auth issues
4. **API Review**: Analyze RESTful conventions, error handling
5. **Middleware Review**: Check middleware patterns and ordering
6. **Generate Report**: Categorize issues by severity

## When to Use

Use `/gin-review` when:
- After writing or modifying Gin handlers
- Before committing Gin route changes
- Reviewing API endpoints
- Adding new middleware
- Onboarding to a Gin codebase

## Review Categories

### CRITICAL (Must Fix)
- Missing authentication on protected routes
- SQL injection in raw queries
- CORS misconfiguration (wildcard origins)
- Input validation missing on user input
- Sensitive data in logs (passwords, tokens)
- Rate limiting missing on public endpoints

### HIGH (Should Fix)
- Using `Bind` instead of `ShouldBind`
- Missing error responses for error conditions
- Handlers over 50 lines (business logic)
- Missing `c.Next()` in middleware
- Wrong HTTP status codes
- Missing route groups for shared middleware

### MEDIUM (Consider)
- Inconsistent RESTful naming
- Missing request ID for tracing
- Using `context.Background()` instead of `c.Request.Context()`
- Duplicate route definitions
- Missing pagination on list endpoints

## Automated Checks Run

```bash
# Static analysis
go vet ./...

# Advanced checks (if installed)
staticcheck ./...
golangci-lint run

# Security scan
govulncheck ./...

# Test with race detection
go test -race ./...
```

## Example Usage

```text
User: /gin-review

Agent:
# Gin Code Review Report

## Files Reviewed
- internal/handler/user.go (modified)
- internal/handler/auth.go (modified)
- internal/middleware/auth.go (modified)
- internal/routes/routes.go (modified)

## Static Analysis Results
✓ go vet: No issues
✓ staticcheck: No issues

## Issues Found

[CRITICAL] Missing Input Validation
File: internal/handler/user.go:28
Issue: Email field not validated
```go
type CreateUserRequest struct {
    Email    string `json:"email"` // Missing validation!
    Password string `json:"password"`
}
```
Fix: Add binding validation
```go
type CreateUserRequest struct {
    Email    string `json:"email" binding:"required,email"`
    Password string `json:"password" binding:"required,min=8"`
}
```

[CRITICAL] CORS Misconfiguration
File: internal/routes/routes.go:15
Issue: AllowOrigins set to wildcard
```go
config := cors.DefaultConfig()
config.AllowOrigins = []string{"*"} // Security risk!
```
Fix: Specify allowed origins
```go
config.AllowOrigins = []string{"https://example.com", "https://api.example.com"}
```

[HIGH] Using Bind Instead of ShouldBind
File: internal/handler/auth.go:42
Issue: Bind sends automatic response, can't customize error
```go
var req LoginRequest
if err := c.Bind(&req); err != nil {
    return // Automatic 400 response
}
```
Fix: Use ShouldBind for custom error handling
```go
var req LoginRequest
if err := c.ShouldBindJSON(&req); err != nil {
    c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
    return
}
```

[HIGH] Handler Bloat
File: internal/handler/order.go:156
Issue: Handler is 87 lines, contains business logic
Fix: Extract business logic to service layer

[MEDIUM] Missing Request ID
File: internal/middleware/logging.go:12
Issue: No request ID for distributed tracing
Fix: Add request ID middleware
```go
func RequestID() gin.HandlerFunc {
    return func(c *gin.Context) {
        requestID := c.GetHeader("X-Request-ID")
        if requestID == "" {
            requestID = uuid.New().String()
        }
        c.Set("request_id", requestID)
        c.Next()
    }
}
```

## Summary
- CRITICAL: 2
- HIGH: 2
- MEDIUM: 1

Recommendation: ❌ Block merge until CRITICAL issues are fixed
```

## Approval Criteria

| Status | Condition |
|--------|-----------|
| ✅ Approve | No CRITICAL or HIGH issues |
| ⚠️ Warning | Only MEDIUM issues (merge with caution) |
| ❌ Block | CRITICAL or HIGH issues found |

## Gin-Specific Checks

### Handler Quality
- [ ] Handlers under 50 lines
- [ ] Business logic extracted to service layer
- [ ] Using `ShouldBind` for validation
- [ ] Proper error responses with status codes

### Middleware Quality
- [ ] `c.Next()` called where appropriate
- [ ] `c.Abort()` before error responses
- [ ] Correct middleware order (CORS → Auth → Logging)
- [ ] Context values properly set/retrieved

### Routing Quality
- [ ] RESTful naming conventions
- [ ] Route groups for shared middleware
- [ ] No duplicate routes
- [ ] Consistent path parameters

### Security
- [ ] Authentication on protected routes
- [ ] Input validation with binding tags
- [ ] CORS properly configured
- [ ] Rate limiting on public endpoints
- [ ] No sensitive data in logs

## Integration with Other Commands

- Use `/gin-test` first to ensure handler tests pass
- Use `/gin-build` if build errors occur
- Use `/gin-review` before committing
- Use `/go-review` for general Go concerns

## Related

- Agent: `agents/gin-reviewer.md`
- Skills: `skills/golang-patterns/`, `skills/api-design/`
