---
description: Enforce TDD workflow for GORM repositories. Write repository tests with mocks or test containers first, then implement. Verify 80%+ coverage.
---

# GORM TDD Command

This command enforces test-driven development methodology for GORM repositories and data access layers.

## What This Command Does

1. **Define Repository Interface**: Scaffold repository methods first
2. **Write Repository Tests**: Create comprehensive test cases (RED)
3. **Run Tests**: Verify tests fail for the right reason
4. **Implement Repository**: Write minimal code to pass (GREEN)
5. **Refactor**: Improve while keeping tests green
6. **Check Coverage**: Ensure 80%+ repository coverage

## When to Use

Use `/gorm-test` when:
- Implementing new GORM repositories
- Adding database query methods
- Testing model associations
- Fixing repository bugs (write failing test first)
- Building critical data access logic

## TDD Cycle

```
RED     → Write failing repository test
GREEN   → Implement minimal repository method
REFACTOR → Optimize queries, tests stay green
REPEAT  → Next test case
```

## Example Session

````
User: /gorm-test I need a repository to find users by email

Agent:
# TDD Session: UserRepository.FindByEmail

## Step 1: Define Repository Interface

```go
// repository/user.go
package repository

import (
    "context"
    "errors"

    "gorm.io/gorm"
    "project/internal/model"
)

var (
    ErrUserNotFound = errors.New("user not found")
)

type UserRepository struct {
    db *gorm.DB
}

func NewUserRepository(db *gorm.DB) *UserRepository {
    return &UserRepository{db: db}
}

// FindByEmail finds a user by email address
func (r *UserRepository) FindByEmail(ctx context.Context, email string) (*model.User, error) {
    panic("not implemented")
}
```

## Step 2: Write Tests (RED)

```go
// repository/user_test.go
package repository

import (
    "context"
    "testing"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
    "gorm.io/driver/sqlite"
    "gorm.io/gorm"
    "project/internal/model"
)

func setupTestDB(t *testing.T) *gorm.DB {
    t.Helper()
    db, err := gorm.Open(sqlite.Open(":memory:"), &gorm.Config{})
    require.NoError(t, err)

    err = db.AutoMigrate(&model.User{})
    require.NoError(t, err)

    t.Cleanup(func() {
        sqlDB, _ := db.DB()
        sqlDB.Close()
    })

    return db
}

func TestUserRepository_FindByEmail(t *testing.T) {
    ctx := context.Background()

    tests := []struct {
        name      string
        seedUsers []*model.User
        email     string
        wantUser  *model.User
        wantErr   error
    }{
        {
            name: "user found",
            seedUsers: []*model.User{
                {Email: "test@example.com", Name: "Test User"},
            },
            email:    "test@example.com",
            wantUser: &model.User{Email: "test@example.com", Name: "Test User"},
            wantErr:  nil,
        },
        {
            name:      "user not found",
            seedUsers: []*model.User{},
            email:     "nonexistent@example.com",
            wantUser:  nil,
            wantErr:   ErrUserNotFound,
        },
        {
            name: "case insensitive email",
            seedUsers: []*model.User{
                {Email: "Test@Example.COM", Name: "Test User"},
            },
            email:    "test@example.com",
            wantUser: &model.User{Email: "Test@Example.COM", Name: "Test User"},
            wantErr:  nil,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            db := setupTestDB(t)
            repo := NewUserRepository(db)

            // Seed data
            for _, u := range tt.seedUsers {
                err := db.Create(u).Error
                require.NoError(t, err)
            }

            // Execute
            got, err := repo.FindByEmail(ctx, tt.email)

            // Assert
            if tt.wantErr != nil {
                assert.ErrorIs(t, err, tt.wantErr)
                assert.Nil(t, got)
            } else {
                assert.NoError(t, err)
                assert.Equal(t, tt.wantUser.Email, got.Email)
                assert.Equal(t, tt.wantUser.Name, got.Name)
                assert.NotZero(t, got.ID)
            }
        })
    }
}
```

## Step 3: Run Tests - Verify FAIL

```bash
$ go test ./repository/...

--- FAIL: TestUserRepository_FindByEmail (0.00s)
    panic: not implemented
FAIL
```

✓ Tests fail as expected.

## Step 4: Implement Repository (GREEN)

```go
// repository/user.go
package repository

import (
    "context"
    "errors"

    "gorm.io/gorm"
    "project/internal/model"
)

var ErrUserNotFound = errors.New("user not found")

type UserRepository struct {
    db *gorm.DB
}

func NewUserRepository(db *gorm.DB) *UserRepository {
    return &UserRepository{db: db}
}

func (r *UserRepository) FindByEmail(ctx context.Context, email string) (*model.User, error) {
    var user model.User
    err := r.db.WithContext(ctx).
        Where("LOWER(email) = LOWER(?)", email).
        First(&user).Error

    if err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, ErrUserNotFound
        }
        return nil, err
    }

    return &user, nil
}
```

## Step 5: Run Tests - Verify PASS

```bash
$ go test ./repository/...

PASS
ok      project/internal/repository   0.025s
```

✓ All tests passing!

## Step 6: Check Coverage

```bash
$ go test -cover ./repository/...

PASS
coverage: 95.0% of statements
ok      project/internal/repository   0.025s
```

✓ Coverage: 95%

## TDD Complete!
````

## Test Patterns

### Test Database Setup
```go
// Using SQLite in-memory for unit tests
func setupTestDB(t *testing.T) *gorm.DB {
    t.Helper()
    db, err := gorm.Open(sqlite.Open(":memory:"), &gorm.Config{})
    require.NoError(t, err)

    err = db.AutoMigrate(models...)
    require.NoError(t, err)

    t.Cleanup(func() {
        sqlDB, _ := db.DB()
        sqlDB.Close()
    })

    return db
}

// Using testcontainers for integration tests
func setupTestContainer(t *testing.T) *gorm.DB {
    ctx := context.Background()
    container, err := testcontainers.Run(ctx, "postgres:15")
    require.NoError(t, err)

    dsn := container.ConnectionString()
    db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{})
    require.NoError(t, err)

    t.Cleanup(func() {
        container.Terminate(ctx)
    })

    return db
}
```

### Repository Interface Testing
```go
type UserRepository interface {
    FindByID(ctx context.Context, id uint) (*model.User, error)
    FindByEmail(ctx context.Context, email string) (*model.User, error)
    Create(ctx context.Context, user *model.User) error
    Update(ctx context.Context, user *model.User) error
    Delete(ctx context.Context, id uint) error
}
```

### Association Testing
```go
func TestUserOrdersAssociation(t *testing.T) {
    db := setupTestDB(t)
    repo := NewUserRepository(db)

    // Create user with orders
    user := &model.User{
        Email: "test@example.com",
        Orders: []model.Order{
            {Total: 100.00},
            {Total: 200.00},
        },
    }

    err := repo.Create(context.Background(), user)
    require.NoError(t, err)

    // Test Preload
    got, err := repo.FindByID(context.Background(), user.ID)
    require.NoError(t, err)
    assert.Len(t, got.Orders, 2)
}
```

### Transaction Testing
```go
func TestCreateOrderInTransaction(t *testing.T) {
    db := setupTestDB(t)
    repo := NewOrderRepository(db)

    order := &model.Order{Total: 100.00}
    items := []model.OrderItem{
        {ProductID: 1, Quantity: 2},
        {ProductID: 2, Quantity: 1},
    }

    err := repo.CreateOrder(context.Background(), order, items)
    require.NoError(t, err)

    // Verify order created
    var count int64
    db.Model(&model.Order{}).Count(&count)
    assert.Equal(t, int64(1), count)

    // Verify items created
    db.Model(&model.OrderItem{}).Count(&count)
    assert.Equal(t, int64(3), count)
}
```

## Coverage Commands

```bash
# Basic coverage
go test -cover ./repository/...

# Coverage profile
go test -coverprofile=coverage.out ./repository/...

# View in browser
go tool cover -html=coverage.out

# Coverage by function
go tool cover -func=coverage.out
```

## Coverage Targets

| Repository Type | Target |
|-----------------|--------|
| Critical data access (auth, payment) | 100% |
| Public repositories | 90%+ |
| General repositories | 80%+ |
| Read-only queries | 70%+ |

## TDD Best Practices

**DO:**
- Write test FIRST, before any implementation
- Use in-memory SQLite for fast unit tests
- Use testcontainers for integration tests
- Test associations with Preload
- Test error conditions (not found, constraint errors)

**DON'T:**
- Use production database for tests
- Skip error path testing
- Ignore transaction rollback scenarios
- Test GORM internals (test your logic)
- Use `time.Sleep` in tests

## Related Commands

- `/gorm-build` - Fix build errors
- `/gorm-review` - Review code after implementation
- `/go-test` - General Go testing

## Related

- Skill: `skills/golang-testing/`
- Agent: `agents/gorm-reviewer.md`
