---
description: Comprehensive GORM code review for model design, associations, query optimization, and database patterns. Invokes the gorm-reviewer agent.
---

# GORM Code Review

This command invokes the **gorm-reviewer** agent for comprehensive GORM-specific code review.

## What This Command Does

1. **Identify GORM Changes**: Find modified model/repository files via `git diff`
2. **Run Static Analysis**: Execute `go vet`, `staticcheck`, and linters
3. **Security Scan**: Check for SQL injection, sensitive data exposure
4. **Model Review**: Analyze model definitions, constraints, indexes
5. **Query Review**: Check for N+1 queries, missing preloads
6. **Generate Report**: Categorize issues by severity

## When to Use

Use `/gorm-review` when:
- After writing or modifying GORM models
- Before committing repository changes
- Reviewing database schema changes
- Adding new associations
- Onboarding to a GORM codebase

## Review Categories

### CRITICAL (Must Fix)
- SQL injection via string concatenation
- Missing transactions for multi-table operations
- Hard deletes on sensitive data (no soft delete)
- Unencrypted passwords/sensitive fields
- Missing constraints (not null, unique) on critical fields

### HIGH (Should Fix)
- N+1 queries (missing Preload)
- Missing indexes on frequently queried columns
- Orphaned records (missing foreign key constraints)
- Race conditions (no optimistic locking)
- Missing CreatedAt/UpdatedAt timestamps

### MEDIUM (Consider)
- Missing context in queries
- Using `Find` when `First` is appropriate
- Not handling `gorm.ErrRecordNotFound`
- Missing soft delete (DeletedAt)
- Inefficient query patterns

## Automated Checks Run

```bash
# Static analysis
go vet ./...

# Advanced checks (if installed)
staticcheck ./...
golangci-lint run

# Security vulnerabilities
govulncheck ./...

# Test with race detection
go test -race ./...
```

## Example Usage

```text
User: /gorm-review

Agent:
# GORM Code Review Report

## Files Reviewed
- internal/model/user.go (modified)
- internal/model/order.go (modified)
- internal/repository/user.go (modified)

## Static Analysis Results
✓ go vet: No issues
✓ staticcheck: No issues

## Issues Found

[CRITICAL] SQL Injection Risk
File: internal/repository/user.go:45
Issue: String concatenation in raw SQL
```go
db.Raw("SELECT * FROM users WHERE email = '" + email + "'").Scan(&user)
```
Fix: Use parameterized query
```go
db.Where("email = ?", email).First(&user)
// Or with Raw
db.Raw("SELECT * FROM users WHERE email = ?", email).Scan(&user)
```

[CRITICAL] Missing Transaction
File: internal/repository/order.go:78
Issue: Multiple database operations without transaction
```go
func CreateOrder(order *Order, items []OrderItem) error {
    db.Create(order)          // Operation 1
    for _, item := range items {
        db.Create(&item)       // Operations 2+
    }
    return nil // What if operation 3 fails?
}
```
Fix: Wrap in transaction
```go
func CreateOrder(order *Order, items []OrderItem) error {
    return db.Transaction(func(tx *gorm.DB) error {
        if err := tx.Create(order).Error; err != nil {
            return err
        }
        for _, item := range items {
            item.OrderID = order.ID
            if err := tx.Create(&item).Error; err != nil {
                return err // Will rollback
            }
        }
        return nil
    })
}
```

[HIGH] N+1 Query Problem
File: internal/repository/user.go:34
Issue: Missing Preload for Orders association
```go
func GetUsers() ([]User, error) {
    var users []User
    db.Find(&users)
    for i := range users {
        db.Model(&users[i]).Association("Orders").Find(&users[i].Orders)
    }
    return users, nil
}
```
Fix: Use Preload
```go
func GetUsers() ([]User, error) {
    var users []User
    err := db.Preload("Orders").Find(&users).Error
    return users, err
}
```

[HIGH] Missing Index
File: internal/model/user.go:15
Issue: Email field queried frequently but not indexed
```go
type User struct {
    ID    uint   `gorm:"primaryKey"`
    Email string `json:"email"` // Missing index!
}
```
Fix: Add index
```go
type User struct {
    ID    uint   `gorm:"primaryKey"`
    Email string `gorm:"uniqueIndex" json:"email"`
}
```

[MEDIUM] Missing Context
File: internal/repository/user.go:56
Issue: Query not using context for cancellation
```go
db.Where("status = ?", "active").Find(&users)
```
Fix: Use WithContext
```go
db.WithContext(ctx).Where("status = ?", "active").Find(&users)
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

## GORM-Specific Checks

### Model Quality
- [ ] Proper gorm tags (primaryKey, uniqueIndex, not null)
- [ ] CreatedAt/UpdatedAt fields present
- [ ] DeletedAt for soft delete (if applicable)
- [ ] TableName() implemented for explicit naming
- [ ] Proper field types for database compatibility

### Association Quality
- [ ] Correct foreign key tags
- [ ] Proper constraint tags (OnDelete, OnUpdate)
- [ ] Many-to-many junction tables defined
- [ ] Pointer types for nullable foreign keys

### Query Quality
- [ ] Preload used for associations
- [ ] Context propagated with WithContext
- [ ] Error handling for ErrRecordNotFound
- [ ] Transactions for multi-operation changes
- [ ] Indexes on frequently queried columns

### Security
- [ ] No string concatenation in queries
- [ ] Sensitive fields not logged
- [ ] Passwords encrypted before storage
- [ ] Soft delete for recoverable data

## Integration with Other Commands

- Use `/gorm-test` first to ensure repository tests pass
- Use `/gorm-build` if build errors occur
- Use `/gorm-review` before committing
- Use `/go-review` for general Go concerns

## Related

- Agent: `agents/gorm-reviewer.md`
- Skills: `skills/golang-patterns/`, `skills/database-patterns/`
