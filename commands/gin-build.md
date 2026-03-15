---
description: Fix Gin framework build errors, routing issues, middleware problems, and binding errors incrementally. Invokes the gin-build-resolver agent.
---

# Gin Build and Fix

This command invokes the **gin-build-resolver** agent to incrementally fix Gin framework errors with minimal changes.

## What This Command Does

1. **Run Diagnostics**: Execute `go build`, check routing conflicts, middleware chains
2. **Parse Errors**: Group by handler/middleware and sort by severity
3. **Fix Incrementally**: One error at a time
4. **Verify Each Fix**: Re-run build after each change
5. **Report Summary**: Show what was fixed and what remains

## When to Use

Use `/gin-build` when:
- Gin routing errors occur (duplicate routes, conflicts)
- Middleware chain issues (missing `c.Next()`, wrong order)
- Binding/validation errors (`Bind` vs `ShouldBind`)
- Context usage errors (nil pointer, goroutine leaks)
- Handler signature mismatches

## Diagnostic Commands Run

```bash
# Primary build check
go build ./...

# Static analysis
go vet ./...

# Check for Gin-specific issues
staticcheck ./...

# Run tests
go test ./...
```

## Example Session

```text
User: /gin-build

Agent:
# Gin Build Resolution

## Initial Diagnostics

```bash
$ go build ./...
internal/handler/user.go:25:15: cannot use handler (type func(http.ResponseWriter, *http.Request)) as type gin.HandlerFunc
internal/handler/user.go:42:9: undefined: c.ShouldBindJSON
internal/middleware/auth.go:18:2: c.Next declared but not called
```

Errors found: 3

## Fix 1: Handler Signature Mismatch

File: internal/handler/user.go:25
Error: cannot use handler as type gin.HandlerFunc
Cause: Using net/http signature instead of gin.Context

```go
// Changed from
func GetUser(w http.ResponseWriter, r *http.Request) {
    // ...
}

// To
func GetUser(c *gin.Context) {
    // ...
}
```

```bash
$ go build ./...
# 2 errors remaining
```

## Fix 2: Undefined Method

File: internal/handler/user.go:42
Error: undefined: c.ShouldBindJSON
Cause: Missing gin import or typo

```go
// Added import
import "github.com/gin-gonic/gin"

// Fixed binding call
var req CreateUserRequest
if err := c.ShouldBindJSON(&req); err != nil {
    c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
    return
}
```

```bash
$ go build ./...
# 1 error remaining
```

## Fix 3: Middleware Missing Next()

File: internal/middleware/auth.go:18
Error: c.Next declared but not called

```go
func AuthMiddleware(c *gin.Context) {
    token := c.GetHeader("Authorization")
    if token == "" {
        c.AbortWithStatusJSON(401, gin.H{"error": "unauthorized"})
        return
    }
    // Validate token...
    c.Next() // Added this line
}
```

```bash
$ go build ./...
# Build successful!
```

## Final Verification

```bash
$ go test ./...
ok      project/internal/handler   0.023s
ok      project/internal/middleware   0.015s
```

## Summary

| Metric | Count |
|--------|-------|
| Build errors fixed | 3 |
| Files modified | 2 |
| Remaining issues | 0 |

Build Status: ✅ SUCCESS
```

## Common Errors Fixed

| Error | Typical Fix |
|-------|-------------|
| `cannot use handler as gin.HandlerFunc` | Change signature to `func(c *gin.Context)` |
| `route conflicts with existing route` | Change path or use route groups |
| `undefined: c.ShouldBind` | Import `github.com/gin-gonic/gin` |
| `c.Next declared but not called` | Add `c.Next()` in middleware |
| `nil pointer dereference` | Check dependency injection |
| `context canceled` | Use `c.Request.Context()` |

## Fix Strategy

1. **Handler signatures** - Ensure all handlers use `func(c *gin.Context)`
2. **Routing conflicts** - Check for duplicate routes
3. **Middleware chain** - Verify `c.Next()` is called
4. **Binding errors** - Use `ShouldBind` over `Bind`
5. **Context usage** - Use `c.Request.Context()` for propagation

## Stop Conditions

The agent will stop and report if:
- Same error persists after 3 attempts
- Fix introduces more errors
- Requires architectural changes
- Missing external dependencies

## Related Commands

- `/gin-review` - Review Gin code quality
- `/gin-test` - Test Gin handlers
- `/go-build` - General Go build issues

## Related

- Agent: `agents/gin-build-resolver.md`
- Skill: `skills/golang-patterns/`
