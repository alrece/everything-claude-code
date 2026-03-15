---
description: Fix GORM model definitions, associations, migrations, and query issues incrementally. Invokes the gorm-build-resolver agent for minimal, surgical fixes.
---

# GORM Build and Fix

This command invokes the **gorm-build-resolver** agent to incrementally fix GORM errors with minimal changes.

## What This Command Does

1. **Run Diagnostics**: Execute `go build`, check model definitions, verify migrations
2. **Parse Errors**: Group by model/repository and sort by severity
3. **Fix Incrementally**: One error at a time
4. **Verify Each Fix**: Re-run build and tests after each change
5. **Report Summary**: Show what was fixed and what remains

## When to Use

Use `/gorm-build` when:
- GORM model definition errors occur
- Association/relationship issues arise
- Migration errors happen
- Query syntax errors appear
- Hook/callback issues occur

## Diagnostic Commands Run

```bash
# Primary build check
go build ./...

# Static analysis
go vet ./...

# Check for GORM-specific issues
staticcheck ./...

# Run tests with verbose output
go test -v ./...
```

## Example Session

```text
User: /gorm-build

Agent:
# GORM Build Resolution

## Initial Diagnostics

```bash
$ go build ./...
internal/model/user.go:25:15: undefined: gorm.DeletedAt
internal/model/order.go:42:9: cannot use Order as type *Order in association
internal/repository/user.go:58:2: undefined: db.Preload
```

Errors found: 3

## Fix 1: Missing GORM Import

File: internal/model/user.go:25
Error: undefined: gorm.DeletedAt
Cause: Missing gorm import

```go
// Added import
import "gorm.io/gorm"

// Model now compiles
type User struct {
    ID        uint           `gorm:"primaryKey"`
    Email     string         `gorm:"uniqueIndex"`
    DeletedAt gorm.DeletedAt `gorm:"index"`
}
```

```bash
$ go build ./...
# 2 errors remaining
```

## Fix 2: Association Type Mismatch

File: internal/model/order.go:42
Error: cannot use Order as type *Order in association
Cause: Pointer/value mismatch in association

```go
// Changed from
type User struct {
    Orders []Order `gorm:"foreignKey:UserID"`
}

// To
type User struct {
    Orders []Order `gorm:"foreignKey:UserID;references:ID"`
}

// Also fixed Order model
type Order struct {
    ID     uint  `gorm:"primaryKey"`
    UserID *uint `gorm:"index"` // Pointer for nullable FK
    User   User  `gorm:"foreignKey:UserID"`
}
```

```bash
$ go build ./...
# 1 error remaining
```

## Fix 3: Missing GORM Method

File: internal/repository/user.go:58
Error: undefined: db.Preload
Cause: Missing gorm.DB import or wrong DB type

```go
// Fixed import
import "gorm.io/gorm"

// Fixed repository
func (r *UserRepository) GetUsersWithOrders(ctx context.Context) ([]User, error) {
    var users []User
    err := r.db.WithContext(ctx).
        Preload("Orders"). // Now works
        Find(&users).Error
    return users, err
}
```

```bash
$ go build ./...
# Build successful!
```

## Final Verification

```bash
$ go test ./...
ok      project/internal/model       0.012s
ok      project/internal/repository  0.025s
```

## Summary

| Metric | Count |
|--------|-------|
| Build errors fixed | 3 |
| Files modified | 3 |
| Remaining issues | 0 |

Build Status: ✅ SUCCESS
```

## Common Errors Fixed

| Error | Typical Fix |
|-------|-------------|
| `undefined: gorm.X` | Import `gorm.io/gorm` |
| `invalid field type` | Use supported types or custom scanner |
| `unknown column` | Add `gorm:"column:name"` tag |
| `table not found` | Implement `TableName()` method |
| `invalid association` | Add `foreignKey` and `references` tags |
| `record not found` in preload | Check association field names |
| `constraint error` | Verify foreign key integrity |
| `duplicate entry` | Add unique constraint or check before insert |

## Fix Strategy

1. **Model definitions** - Ensure proper gorm tags and types
2. **Associations** - Verify foreign keys and references
3. **Migrations** - Check AutoMigrate calls
4. **Queries** - Fix method chains and syntax
5. **Hooks** - Ensure correct signatures

## Stop Conditions

The agent will stop and report if:
- Same error persists after 3 attempts
- Fix introduces more errors
- Requires schema changes beyond scope
- Missing external dependencies

## Related Commands

- `/gorm-review` - Review GORM code quality
- `/gorm-test` - Test GORM repositories
- `/go-build` - General Go build issues

## Related

- Agent: `agents/gorm-build-resolver.md`
- Skill: `skills/golang-patterns/`
