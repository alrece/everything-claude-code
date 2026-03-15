---
paths:
  - "**/*.go"
  - "**/go.mod"
---
# GORM Coding Style

> This file extends [common/coding-style.md](../common/coding-style.md) with GORM-specific content.

## Model Definition

Use complete GORM tags and follow naming conventions:

```go
// GOOD: Complete model with proper tags
type User struct {
    ID        uint           `gorm:"primaryKey" json:"id"`
    Email     string         `gorm:"size:255;uniqueIndex;not null" json:"email"`
    Name      string         `gorm:"size:100;not null" json:"name"`
    Password  string         `gorm:"size:255;not null" json:"-"` // Never expose sensitive fields
    Status    string         `gorm:"size:20;default:'active';index" json:"status"`
    CreatedAt time.Time      `json:"createdAt"`
    UpdatedAt time.Time      `json:"updatedAt"`
    DeletedAt gorm.DeletedAt `gorm:"index" json:"-"`
}

// BAD: Missing tags and constraints
type User struct {
    ID       int
    Email    string
    Name     string
    Password string
}
```

## Table Naming

Implement `TableName()` for explicit table names:

```go
// GOOD: Explicit table name
func (User) TableName() string {
    return "users"
}

// GOOD: With schema
func (User) TableName() string {
    return "public.users"
}

// Avoid relying on default pluralization
```

## Field Types

Use appropriate field types for database compatibility:

```go
type Order struct {
    ID          uint          `gorm:"primaryKey"`
    TotalAmount float64       `gorm:"type:decimal(10,2);not null"` // Use decimal for money
    Quantity    int           `gorm:"not null;default:0"`
    Description string        `gorm:"type:text"` // Use text for long strings
    Status      string        `gorm:"size:20;not null;default:'pending'"`
    Metadata    datatypes.JSON `gorm:"type:jsonb"` // For PostgreSQL JSON
    CreatedAt   time.Time
}
```

## Association Definition

Define associations with explicit foreign keys:

```go
// GOOD: Explicit foreign keys and constraints
type User struct {
    ID      uint     `gorm:"primaryKey"`
    Profile Profile  `gorm:"foreignKey:UserID;constraint:OnDelete:CASCADE"`
    Orders  []Order  `gorm:"foreignKey:UserID;constraint:OnDelete:SET NULL"`
    Roles   []Role   `gorm:"many2many:user_roles;"`
}

type Order struct {
    ID          uint        `gorm:"primaryKey"`
    UserID      *uint       `gorm:"index"` // Pointer for nullable FK
    User        User        `gorm:"foreignKey:UserID"`
    OrderItems  []OrderItem `gorm:"foreignKey:OrderID"`
}

// BAD: Missing constraints
type User struct {
    ID      uint
    Orders  []Order // No foreign key definition
}
```

## Repository Pattern

Use repository pattern for data access:

```go
type UserRepository interface {
    Create(ctx context.Context, user *User) error
    GetByID(ctx context.Context, id uint) (*User, error)
    GetByEmail(ctx context.Context, email string) (*User, error)
    Update(ctx context.Context, user *User) error
    Delete(ctx context.Context, id uint) error
    List(ctx context.Context, offset, limit int) ([]User, int64, error)
}

type userRepository struct {
    db *gorm.DB
}

func NewUserRepository(db *gorm.DB) UserRepository {
    return &userRepository{db: db}
}

func (r *userRepository) GetByID(ctx context.Context, id uint) (*User, error) {
    var user User
    err := r.db.WithContext(ctx).First(&user, id).Error
    if err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, ErrUserNotFound
        }
        return nil, err
    }
    return &user, nil
}
```

## Context Propagation

Always use `WithContext`:

```go
// GOOD: Context propagated
func (r *userRepository) GetByID(ctx context.Context, id uint) (*User, error) {
    var user User
    err := r.db.WithContext(ctx).First(&user, id).Error
    return &user, err
}

// BAD: No context
func (r *userRepository) GetByID(id uint) (*User, error) {
    var user User
    err := r.db.First(&user, id).Error
    return &user, err
}
```

## Error Handling

Handle GORM errors properly:

```go
func (r *userRepository) GetByID(ctx context.Context, id uint) (*User, error) {
    var user User
    err := r.db.WithContext(ctx).First(&user, id).Error
    if err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, ErrUserNotFound // Custom error
        }
        return nil, fmt.Errorf("failed to get user: %w", err)
    }
    return &user, nil
}
```

## Query Building

Chain methods for readable queries:

```go
// GOOD: Clear query chain
func (r *userRepository) Search(ctx context.Context, params SearchParams) ([]User, int64, error) {
    query := r.db.WithContext(ctx).Model(&User{})

    if params.Email != "" {
        query = query.Where("email LIKE ?", "%"+params.Email+"%")
    }
    if params.Status != "" {
        query = query.Where("status = ?", params.Status)
    }

    var total int64
    query.Count(&total)

    var users []User
    err := query.Offset(params.Offset).Limit(params.Limit).Find(&users).Error
    return users, total, err
}
```
