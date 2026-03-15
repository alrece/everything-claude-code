---
paths:
  - "**/*.go"
  - "**/*_test.go"
---
# GORM Testing

> This file extends [common/testing.md](../common/testing.md) with GORM-specific content.

## Test Database Setup

Use in-memory SQLite for unit tests:

```go
import (
    "testing"
    "gorm.io/driver/sqlite"
    "gorm.io/gorm"
)

func setupTestDB(t *testing.T) *gorm.DB {
    t.Helper()

    db, err := gorm.Open(sqlite.Open(":memory:"), &gorm.Config{})
    if err != nil {
        t.Fatalf("Failed to connect to test database: %v", err)
    }

    // Run migrations
    err = db.AutoMigrate(&User{}, &Order{}, &OrderItem{})
    if err != nil {
        t.Fatalf("Failed to migrate: %v", err)
    }

    t.Cleanup(func() {
        sqlDB, _ := db.DB()
        sqlDB.Close()
    })

    return db
}
```

## Repository Testing

```go
func TestUserRepository_Create(t *testing.T) {
    db := setupTestDB(t)
    repo := NewUserRepository(db)
    ctx := context.Background()

    user := &User{
        Email:    "test@example.com",
        Name:     "Test User",
        Password: "hashedpassword",
    }

    err := repo.Create(ctx, user)
    assert.NoError(t, err)
    assert.NotZero(t, user.ID)

    // Verify created
    found, err := repo.GetByID(ctx, user.ID)
    assert.NoError(t, err)
    assert.Equal(t, user.Email, found.Email)
}

func TestUserRepository_GetByEmail_NotFound(t *testing.T) {
    db := setupTestDB(t)
    repo := NewUserRepository(db)
    ctx := context.Background()

    _, err := repo.GetByEmail(ctx, "nonexistent@example.com")
    assert.ErrorIs(t, err, ErrUserNotFound)
}
```

## Table-Driven Tests

```go
func TestUserRepository_List(t *testing.T) {
    tests := []struct {
        name       string
        seedUsers  []User
        params     ListParams
        wantCount  int
        wantTotal  int64
    }{
        {
            name: "empty list",
            seedUsers: []User{},
            params: ListParams{Page: 1, PageSize: 10},
            wantCount: 0,
            wantTotal: 0,
        },
        {
            name: "single page",
            seedUsers: []User{
                {Email: "user1@example.com", Name: "User 1"},
                {Email: "user2@example.com", Name: "User 2"},
            },
            params: ListParams{Page: 1, PageSize: 10},
            wantCount: 2,
            wantTotal: 2,
        },
        {
            name: "pagination",
            seedUsers: []User{
                {Email: "user1@example.com", Name: "User 1"},
                {Email: "user2@example.com", Name: "User 2"},
                {Email: "user3@example.com", Name: "User 3"},
            },
            params: ListParams{Page: 1, PageSize: 2},
            wantCount: 2,
            wantTotal: 3,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            db := setupTestDB(t)
            repo := NewUserRepository(db)
            ctx := context.Background()

            // Seed data
            for _, u := range tt.seedUsers {
                repo.Create(ctx, &u)
            }

            // Execute
            users, total, err := repo.List(ctx, tt.params)

            // Assert
            assert.NoError(t, err)
            assert.Len(t, users, tt.wantCount)
            assert.Equal(t, tt.wantTotal, total)
        })
    }
}
```

## Transaction Testing

```go
func TestOrderService_CreateOrder_Transaction(t *testing.T) {
    db := setupTestDB(t)

    // Seed product with limited stock
    product := &Product{Name: "Test Product", Stock: 10}
    db.Create(product)

    svc := NewOrderService(db)
    ctx := context.Background()

    t.Run("successful order", func(t *testing.T) {
        order := &Order{UserID: 1}
        items := []OrderItem{
            {ProductID: product.ID, Quantity: 5},
        }

        err := svc.CreateOrder(ctx, order, items)
        assert.NoError(t, err)

        // Verify stock deducted
        var updated Product
        db.First(&updated, product.ID)
        assert.Equal(t, 5, updated.Stock)
    })

    t.Run("insufficient stock rollback", func(t *testing.T) {
        order := &Order{UserID: 1}
        items := []OrderItem{
            {ProductID: product.ID, Quantity: 100}, // More than available
        }

        err := svc.CreateOrder(ctx, order, items)
        assert.Error(t, err)

        // Verify no order created
        var count int64
        db.Model(&Order{}).Count(&count)
        assert.Equal(t, int64(0), count)
    })
}
```

## Association Testing

```go
func TestUserRepository_GetWithOrders(t *testing.T) {
    db := setupTestDB(t)
    repo := NewUserRepository(db)
    ctx := context.Background()

    // Create user with orders
    user := &User{
        Email: "test@example.com",
        Name:  "Test User",
        Orders: []Order{
            {TotalAmount: 100.00},
            {TotalAmount: 200.00},
        },
    }

    err := repo.Create(ctx, user)
    require.NoError(t, err)

    // Test preload
    found, err := repo.GetWithOrders(ctx, user.ID)
    require.NoError(t, err)

    assert.Len(t, found.Orders, 2)
    assert.Equal(t, 100.00, found.Orders[0].TotalAmount)
}
```

## Mock Repository

```go
type MockUserRepository struct {
    CreateFunc   func(ctx context.Context, user *User) error
    GetByIDFunc  func(ctx context.Context, id uint) (*User, error)
    GetByEmailFunc func(ctx context.Context, email string) (*User, error)
    UpdateFunc   func(ctx context.Context, user *User) error
    DeleteFunc   func(ctx context.Context, id uint) error
    ListFunc     func(ctx context.Context, params ListParams) ([]User, int64, error)
}

func (m *MockUserRepository) Create(ctx context.Context, user *User) error {
    if m.CreateFunc != nil {
        return m.CreateFunc(ctx, user)
    }
    return errors.New("not implemented")
}

func (m *MockUserRepository) GetByID(ctx context.Context, id uint) (*User, error) {
    if m.GetByIDFunc != nil {
        return m.GetByIDFunc(ctx, id)
    }
    return nil, errors.New("not implemented")
}

// ... other methods
```

## Test Helpers

```go
func seedTestData(t *testing.T, db *gorm.DB, models ...interface{}) {
    t.Helper()
    for _, model := range models {
        if err := db.Create(model).Error; err != nil {
            t.Fatalf("Failed to seed test data: %v", err)
        }
    }
}

func assertRecordExists(t *testing.T, db *gorm.DB, model interface{}, conditions ...interface{}) {
    t.Helper()
    result := db.First(model, conditions...)
    assert.NoError(t, result.Error, "Expected record to exist")
}

func assertRecordNotExists(t *testing.T, db *gorm.DB, model interface{}, conditions ...interface{}) {
    t.Helper()
    result := db.First(model, conditions...)
    assert.ErrorIs(t, result.Error, gorm.ErrRecordNotFound, "Expected record to not exist")
}

func countRecords(t *testing.T, db *gorm.DB, model interface{}) int64 {
    t.Helper()
    var count int64
    db.Model(model).Count(&count)
    return count
}
```

## Integration Tests with Testcontainers

```go
// +build integration

func TestUserRepository_Integration(t *testing.T) {
    if testing.Short() {
        t.Skip("Skipping integration test")
    }

    ctx := context.Background()

    // Start PostgreSQL container
    container, err := testcontainers.Run(ctx, "postgres:15",
        testcontainers.WithEnv(map[string]string{
            "POSTGRES_USER":     "test",
            "POSTGRES_PASSWORD": "test",
            "POSTGRES_DB":       "testdb",
        }),
    )
    require.NoError(t, err)
    defer container.Terminate(ctx)

    // Get connection string
    dsn, err := container.ConnectionString(ctx)
    require.NoError(t, err)

    // Connect
    db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{})
    require.NoError(t, err)

    // Run tests...
}
```

## Coverage Targets

| Component | Target |
|-----------|--------|
| Repository CRUD | 95%+ |
| Custom queries | 90%+ |
| Transactions | 100% |
| Error handling | 100% |

```bash
# Run tests with coverage
go test -cover ./repository/...

# Generate coverage report
go test -coverprofile=coverage.out ./repository/...
go tool cover -html=coverage.out
```
