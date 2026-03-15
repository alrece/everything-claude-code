---
paths:
  - "**/*.go"
  - "**/go.mod"
---
# GORM Patterns

> This file extends [common/patterns.md](../common/patterns.md) with GORM-specific content.

## Repository Pattern

```go
type UserRepository interface {
    Create(ctx context.Context, user *User) error
    GetByID(ctx context.Context, id uint) (*User, error)
    GetByEmail(ctx context.Context, email string) (*User, error)
    Update(ctx context.Context, user *User) error
    Delete(ctx context.Context, id uint) error
    List(ctx context.Context, params ListParams) ([]User, int64, error)
}

type userRepository struct {
    db *gorm.DB
}

func NewUserRepository(db *gorm.DB) UserRepository {
    return &userRepository{db: db}
}
```

## Preload Pattern

Avoid N+1 queries with Preload:

```go
// GOOD: Use Preload for associations
func (r *userRepository) GetWithOrders(ctx context.Context, id uint) (*User, error) {
    var user User
    err := r.db.WithContext(ctx).
        Preload("Orders").
        Preload("Orders.Items").
        First(&user, id).Error
    return &user, err
}

// GOOD: Conditional Preload
func (r *userRepository) GetWithActiveOrders(ctx context.Context, id uint) (*User, error) {
    var user User
    err := r.db.WithContext(ctx).
        Preload("Orders", "status = ?", "active").
        First(&user, id).Error
    return &user, err
}

// BAD: N+1 query
func (r *userRepository) GetWithOrders(ctx context.Context, id uint) (*User, error) {
    var user User
    r.db.First(&user, id)
    r.db.Model(&user).Association("Orders").Find(&user.Orders)
    return &user, nil
}
```

## Transaction Pattern

Use transactions for multi-operation changes:

```go
// GOOD: Transaction with callback
func (s *OrderService) CreateOrder(ctx context.Context, order *Order, items []OrderItem) error {
    return s.db.Transaction(func(tx *gorm.DB) error {
        // Create order
        if err := tx.Create(order).Error; err != nil {
            return err // Will rollback
        }

        // Create order items
        for i := range items {
            items[i].OrderID = order.ID
            if err := tx.Create(&items[i]).Error; err != nil {
                return err // Will rollback
            }
        }

        // Update inventory
        for _, item := range items {
            if err := tx.Model(&Product{}).
                Where("id = ? AND stock >= ?", item.ProductID, item.Quantity).
                Update("stock", gorm.Expr("stock - ?", item.Quantity)).Error; err != nil {
                return err // Will rollback
            }
        }

        return nil // Commit
    })
}

// Manual transaction control
func (s *OrderService) CreateOrderManual(ctx context.Context, order *Order) error {
    tx := s.db.Begin()
    defer func() {
        if r := recover(); r != nil {
            tx.Rollback()
        }
    }()

    if err := tx.Create(order).Error; err != nil {
        tx.Rollback()
        return err
    }

    return tx.Commit().Error
}
```

## Soft Delete Pattern

Use soft delete for recoverable data:

```go
type User struct {
    ID        uint           `gorm:"primaryKey"`
    Name      string         `gorm:"not null"`
    DeletedAt gorm.DeletedAt `gorm:"index"`
}

// Soft delete (sets deleted_at)
db.Delete(&user, id)

// Find excludes soft-deleted records
db.Find(&users)

// Include soft-deleted records
db.Unscoped().Find(&users)

// Permanent delete
db.Unscoped().Delete(&user, id)
```

## Hooks Pattern

Use hooks for validation and side effects:

```go
type User struct {
    ID       uint   `gorm:"primaryKey"`
    Email    string `gorm:"not null"`
    Password string `gorm:"not null"`
    UUID     string `gorm:"uniqueIndex"`
}

// Before create hook
func (u *User) BeforeCreate(tx *gorm.DB) error {
    if u.UUID == "" {
        u.UUID = uuid.New().String()
    }

    // Validate email
    if !isValidEmail(u.Email) {
        return errors.New("invalid email format")
    }

    return nil
}

// Before update hook
func (u *User) BeforeUpdate(tx *gorm.DB) error {
    // Prevent email change
    if tx.Statement.Changed("Email") {
        return errors.New("email cannot be changed")
    }
    return nil
}

// After create hook
func (u *User) AfterCreate(tx *gorm.DB) error {
    // Send welcome email (async)
    go sendWelcomeEmail(u.Email)
    return nil
}
```

## Scope Pattern

Use scopes for reusable query logic:

```go
// Define scopes
func WithStatus(status string) func(*gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        return db.Where("status = ?", status)
    }
}

func CreatedAfter(t time.Time) func(*gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        return db.Where("created_at > ?", t)
    }
}

func Paginate(page, pageSize int) func(*gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        offset := (page - 1) * pageSize
        return db.Offset(offset).Limit(pageSize)
    }
}

// Use scopes
func (r *userRepository) ListActive(ctx context.Context, page, pageSize int) ([]User, error) {
    var users []User
    err := r.db.WithContext(ctx).
        Scopes(
            WithStatus("active"),
            Paginate(page, pageSize),
        ).
        Find(&users).Error
    return users, err
}
```

## Batch Operations

Use batch operations for efficiency:

```go
// Batch insert
users := []User{
    {Name: "User 1", Email: "user1@example.com"},
    {Name: "User 2", Email: "user2@example.com"},
}
db.CreateInBatches(users, 100) // Insert 100 records per batch

// Batch update
db.Model(&User{}).
    Where("status = ?", "pending").
    Update("status", "processed")

// Batch delete
db.Where("created_at < ?", time.Now().AddDate(-1, 0, 0)).
    Delete(&User{})
```

## Optimistic Locking

Implement optimistic locking for concurrent updates:

```go
type Product struct {
    ID      uint   `gorm:"primaryKey"`
    Name    string `gorm:"not null"`
    Stock   int    `gorm:"not null"`
    Version int    `gorm:"not null;default:0"`
}

func (r *ProductRepository) UpdateStock(ctx context.Context, id uint, quantity int) error {
    result := r.db.WithContext(ctx).
        Model(&Product{}).
        Where("id = ? AND version = ?", id, product.Version).
        Updates(map[string]interface{}{
            "stock":   gorm.Expr("stock - ?", quantity),
            "version": gorm.Expr("version + 1"),
        })

    if result.Error != nil {
        return result.Error
    }

    if result.RowsAffected == 0 {
        return ErrConcurrentUpdate // Optimistic lock failed
    }

    return nil
}
```

## References

See skill: `golang-patterns` for detailed Go patterns.
See skill: `gorm-best-practices` for advanced GORM patterns.
