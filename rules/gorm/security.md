---
paths:
  - "**/*.go"
  - "**/go.mod"
---
# GORM Security

> This file extends [common/security.md](../common/security.md) with GORM-specific content.

## SQL Injection Prevention

Never concatenate user input in queries:

```go
// BAD: SQL injection vulnerability
email := c.Query("email")
query := fmt.Sprintf("SELECT * FROM users WHERE email = '%s'", email)
db.Raw(query).Scan(&users)

// GOOD: Parameterized query
email := c.Query("email")
db.Where("email = ?", email).Find(&users)

// GOOD: Parameterized raw SQL
db.Raw("SELECT * FROM users WHERE email = ?", email).Scan(&users)

// GOOD: Named parameters
db.Where("email = @email", sql.Named("email", email)).Find(&users)
```

## Input Validation

Validate all input before database operations:

```go
type CreateUserRequest struct {
    Email    string `json:"email" binding:"required,email"`
    Password string `json:"password" binding:"required,min=8"`
    Name     string `json:"name" binding:"required,min=2,max=100"`
}

func (s *UserService) Create(ctx context.Context, req CreateUserRequest) (*User, error) {
    // Input is already validated by Gin binding
    user := &User{
        Email:    req.Email,
        Name:     req.Name,
        Password: hashPassword(req.Password),
    }

    if err := s.repo.Create(ctx, user); err != nil {
        return nil, err
    }

    return user, nil
}
```

## Sensitive Data Handling

Never expose sensitive fields:

```go
type User struct {
    ID        uint           `gorm:"primaryKey" json:"id"`
    Email     string         `gorm:"size:255" json:"email"`
    Password  string         `gorm:"size:255" json:"-"`        // Never expose
    Salt      string         `gorm:"size:32" json:"-"`         // Never expose
    Token     string         `gorm:"size:255" json:"-"`        // Never expose
    CreatedAt time.Time      `json:"createdAt"`
    DeletedAt gorm.DeletedAt `gorm:"index" json:"-"`
}

// Use separate response type
type UserResponse struct {
    ID        uint   `json:"id"`
    Email     string `json:"email"`
    Name      string `json:"name"`
    CreatedAt string `json:"createdAt"`
}

func (s *UserService) GetByID(ctx context.Context, id uint) (*UserResponse, error) {
    user, err := s.repo.GetByID(ctx, id)
    if err != nil {
        return nil, err
    }

    return &UserResponse{
        ID:        user.ID,
        Email:     user.Email,
        Name:      user.Name,
        CreatedAt: user.CreatedAt.Format(time.RFC3339),
    }, nil
}
```

## Password Storage

Always hash passwords before storage:

```go
import "golang.org/x/crypto/bcrypt"

func hashPassword(password string) string {
    hash, _ := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
    return string(hash)
}

func verifyPassword(password, hash string) bool {
    err := bcrypt.CompareHashAndPassword([]byte(hash), []byte(password))
    return err == nil
}

// NEVER store plain text passwords
// BAD:
user.Password = password
// GOOD:
user.Password = hashPassword(password)
```

## Transaction Security

Use transactions for data integrity:

```go
// GOOD: Transaction ensures atomicity
func (s *PaymentService) ProcessPayment(ctx context.Context, payment *Payment) error {
    return s.db.Transaction(func(tx *gorm.DB) error {
        // Deduct balance
        result := tx.Model(&Account{}).
            Where("id = ? AND balance >= ?", payment.AccountID, payment.Amount).
            Update("balance", gorm.Expr("balance - ?", payment.Amount))

        if result.Error != nil {
            return result.Error
        }

        if result.RowsAffected == 0 {
            return ErrInsufficientBalance
        }

        // Create payment record
        if err := tx.Create(payment).Error; err != nil {
            return err
        }

        return nil
    })
}

// BAD: Multiple operations without transaction
func (s *PaymentService) ProcessPayment(payment *Payment) error {
    // Deduct balance
    s.db.Model(&Account{}).Update("balance", gorm.Expr("balance - ?", payment.Amount))
    // What if this fails?
    s.db.Create(payment)
    return nil
}
```

## Query Scope Limitation

Limit query results to prevent data leakage:

```go
// GOOD: Use WHERE to limit scope
func (r *OrderRepository) GetUserOrders(ctx context.Context, userID uint) ([]Order, error) {
    var orders []Order
    err := r.db.WithContext(ctx).
        Where("user_id = ?", userID). // Scope to user
        Find(&orders).Error
    return orders, err
}

// GOOD: Use Select to limit fields
func (r *UserRepository) GetPublicProfile(ctx context.Context, id uint) (*UserPublicProfile, error) {
    var profile UserPublicProfile
    err := r.db.WithContext(ctx).
        Model(&User{}).
        Select("id, name, avatar"). // Only public fields
        First(&profile, id).Error
    return &profile, err
}

// BAD: Returns all data
func (r *OrderRepository) GetAllOrders() ([]Order, error) {
    var orders []Order
    r.db.Find(&orders) // Returns ALL orders
    return orders, nil
}
```

## Soft Delete for Sensitive Data

Use soft delete to preserve data:

```go
type User struct {
    ID        uint           `gorm:"primaryKey"`
    Email     string         `gorm:"uniqueIndex"`
    DeletedAt gorm.DeletedAt `gorm:"index"`
}

// Soft delete preserves data for audit/recovery
db.Delete(&user, id)

// Hard delete only when explicitly required
// db.Unscoped().Delete(&user, id)
```

## Logging Sensitive Data

Never log sensitive query data:

```go
// BAD: Logging with sensitive data
func (r *UserRepository) GetByEmail(email string) (*User, error) {
    log.Printf("Getting user by email: %s", email)
    // ...
}

// GOOD: Log without sensitive data
func (r *UserRepository) GetByEmail(ctx context.Context, email string) (*User, error) {
    requestID := ctx.Value("request_id")
    log.Printf("[%s] Getting user by email", requestID)
    // ...
}

// Configure GORM logger to exclude sensitive data
db, _ := gorm.Open(postgres.Open(dsn), &gorm.Config{
    Logger: logger.Default.LogMode(logger.Info),
})
```

## Database Connection Security

Secure database connection:

```go
// GOOD: Use environment variables
dsn := fmt.Sprintf(
    "host=%s user=%s password=%s dbname=%s port=%s sslmode=require",
    os.Getenv("DB_HOST"),
    os.Getenv("DB_USER"),
    os.Getenv("DB_PASSWORD"),
    os.Getenv("DB_NAME"),
    os.Getenv("DB_PORT"),
)

// BAD: Hardcoded credentials
dsn := "host=localhost user=admin password=admin123 dbname=mydb"

// GOOD: Use connection pooling
sqlDB, _ := db.DB()
sqlDB.SetMaxIdleConns(10)
sqlDB.SetMaxOpenConns(100)
sqlDB.SetConnMaxLifetime(time.Hour)
```
