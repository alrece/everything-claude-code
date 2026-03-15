---
paths:
  - "**/*.go"
  - "**/go.mod"
---
# GORM 安全

> 此文件扩展了 [common/security.md](../common/security.md) 的 GORM 特定内容。

## SQL 注入防护

永远不要在查询中拼接用户输入：

```go
// 错误：SQL 注入漏洞
email := c.Query("email")
query := fmt.Sprintf("SELECT * FROM users WHERE email = '%s'", email)
db.Raw(query).Scan(&users)

// 正确：参数化查询
email := c.Query("email")
db.Where("email = ?", email).Find(&users)

// 正确：参数化原生 SQL
db.Raw("SELECT * FROM users WHERE email = ?", email).Scan(&users)

// 正确：命名参数
db.Where("email = @email", sql.Named("email", email)).Find(&users)
```

## 输入验证

数据库操作前验证所有输入：

```go
type CreateUserRequest struct {
    Email    string `json:"email" binding:"required,email"`
    Password string `json:"password" binding:"required,min=8"`
    Name     string `json:"name" binding:"required,min=2,max=100"`
}

func (s *UserService) Create(ctx context.Context, req CreateUserRequest) (*User, error) {
    // 输入已通过 Gin binding 验证
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

## 敏感数据处理

永远不要暴露敏感字段：

```go
type User struct {
    ID        uint           `gorm:"primaryKey" json:"id"`
    Email     string         `gorm:"size:255" json:"email"`
    Password  string         `gorm:"size:255" json:"-"`        // 永不暴露
    Salt      string         `gorm:"size:32" json:"-"`         // 永不暴露
    Token     string         `gorm:"size:255" json:"-"`        // 永不暴露
    CreatedAt time.Time      `json:"createdAt"`
    DeletedAt gorm.DeletedAt `gorm:"index" json:"-"`
}

// 使用单独的响应类型
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

## 密码存储

存储前始终哈希密码：

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

// 永远不要存储明文密码
// 错误：
user.Password = password
// 正确：
user.Password = hashPassword(password)
```

## 事务安全

使用事务确保数据完整性：

```go
// 正确：事务确保原子性
func (s *PaymentService) ProcessPayment(ctx context.Context, payment *Payment) error {
    return s.db.Transaction(func(tx *gorm.DB) error {
        // 扣除余额
        result := tx.Model(&Account{}).
            Where("id = ? AND balance >= ?", payment.AccountID, payment.Amount).
            Update("balance", gorm.Expr("balance - ?", payment.Amount))

        if result.Error != nil {
            return result.Error
        }

        if result.RowsAffected == 0 {
            return ErrInsufficientBalance
        }

        // 创建支付记录
        if err := tx.Create(payment).Error; err != nil {
            return err
        }

        return nil
    })
}

// 错误：没有事务的多个操作
func (s *PaymentService) ProcessPayment(payment *Payment) error {
    // 扣除余额
    s.db.Model(&Account{}).Update("balance", gorm.Expr("balance - ?", payment.Amount))
    // 如果这里失败怎么办？
    s.db.Create(payment)
    return nil
}
```

## 查询范围限制

限制查询结果以防止数据泄露：

```go
// 正确：使用 WHERE 限制范围
func (r *OrderRepository) GetUserOrders(ctx context.Context, userID uint) ([]Order, error) {
    var orders []Order
    err := r.db.WithContext(ctx).
        Where("user_id = ?", userID). // 限制到用户
        Find(&orders).Error
    return orders, err
}

// 正确：使用 Select 限制字段
func (r *UserRepository) GetPublicProfile(ctx context.Context, id uint) (*UserPublicProfile, error) {
    var profile UserPublicProfile
    err := r.db.WithContext(ctx).
        Model(&User{}).
        Select("id, name, avatar"). // 仅公共字段
        First(&profile, id).Error
    return &profile, err
}

// 错误：返回所有数据
func (r *OrderRepository) GetAllOrders() ([]Order, error) {
    var orders []Order
    r.db.Find(&orders) // 返回所有订单
    return orders, nil
}
```

## 敏感数据软删除

使用软删除保留数据：

```go
type User struct {
    ID        uint           `gorm:"primaryKey"`
    Email     string         `gorm:"uniqueIndex"`
    DeletedAt gorm.DeletedAt `gorm:"index"`
}

// 软删除保留数据用于审计/恢复
db.Delete(&user, id)

// 仅在明确需要时硬删除
// db.Unscoped().Delete(&user, id)
```

## 记录敏感数据

永远不要记录敏感查询数据：

```go
// 错误：记录敏感数据
func (r *UserRepository) GetByEmail(email string) (*User, error) {
    log.Printf("根据邮箱获取用户: %s", email)
    // ...
}

// 正确：不记录敏感数据
func (r *UserRepository) GetByEmail(ctx context.Context, email string) (*User, error) {
    requestID := ctx.Value("request_id")
    log.Printf("[%s] 根据邮箱获取用户", requestID)
    // ...
}

// 配置 GORM 日志排除敏感数据
db, _ := gorm.Open(postgres.Open(dsn), &gorm.Config{
    Logger: logger.Default.LogMode(logger.Info),
})
```

## 数据库连接安全

安全数据库连接：

```go
// 正确：使用环境变量
dsn := fmt.Sprintf(
    "host=%s user=%s password=%s dbname=%s port=%s sslmode=require",
    os.Getenv("DB_HOST"),
    os.Getenv("DB_USER"),
    os.Getenv("DB_PASSWORD"),
    os.Getenv("DB_NAME"),
    os.Getenv("DB_PORT"),
)

// 错误：硬编码凭据
dsn := "host=localhost user=admin password=admin123 dbname=mydb"

// 正确：使用连接池
sqlDB, _ := db.DB()
sqlDB.SetMaxIdleConns(10)
sqlDB.SetMaxOpenConns(100)
sqlDB.SetConnMaxLifetime(time.Hour)
```
