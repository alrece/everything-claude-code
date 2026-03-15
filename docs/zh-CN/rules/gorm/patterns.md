---
paths:
  - "**/*.go"
  - "**/go.mod"
---
# GORM 模式

> 此文件扩展了 [common/patterns.md](../common/patterns.md) 的 GORM 特定内容。

## 仓库模式

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

## Preload 模式

使用 Preload 避免 N+1 查询：

```go
// 正确：使用 Preload 加载关联
func (r *userRepository) GetWithOrders(ctx context.Context, id uint) (*User, error) {
    var user User
    err := r.db.WithContext(ctx).
        Preload("Orders").
        Preload("Orders.Items").
        First(&user, id).Error
    return &user, err
}

// 正确：条件 Preload
func (r *userRepository) GetWithActiveOrders(ctx context.Context, id uint) (*User, error) {
    var user User
    err := r.db.WithContext(ctx).
        Preload("Orders", "status = ?", "active").
        First(&user, id).Error
    return &user, err
}

// 错误：N+1 查询
func (r *userRepository) GetWithOrders(ctx context.Context, id uint) (*User, error) {
    var user User
    r.db.First(&user, id)
    r.db.Model(&user).Association("Orders").Find(&user.Orders)
    return &user, nil
}
```

## 事务模式

多操作变更使用事务：

```go
// 正确：带回调的事务
func (s *OrderService) CreateOrder(ctx context.Context, order *Order, items []OrderItem) error {
    return s.db.Transaction(func(tx *gorm.DB) error {
        // 创建订单
        if err := tx.Create(order).Error; err != nil {
            return err // 将回滚
        }

        // 创建订单项
        for i := range items {
            items[i].OrderID = order.ID
            if err := tx.Create(&items[i]).Error; err != nil {
                return err // 将回滚
            }
        }

        // 更新库存
        for _, item := range items {
            if err := tx.Model(&Product{}).
                Where("id = ? AND stock >= ?", item.ProductID, item.Quantity).
                Update("stock", gorm.Expr("stock - ?", item.Quantity)).Error; err != nil {
                return err // 将回滚
            }
        }

        return nil // 提交
    })
}

// 手动事务控制
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

## 软删除模式

可恢复数据使用软删除：

```go
type User struct {
    ID        uint           `gorm:"primaryKey"`
    Name      string         `gorm:"not null"`
    DeletedAt gorm.DeletedAt `gorm:"index"`
}

// 软删除（设置 deleted_at）
db.Delete(&user, id)

// Find 排除软删除记录
db.Find(&users)

// 包含软删除记录
db.Unscoped().Find(&users)

// 永久删除
db.Unscoped().Delete(&user, id)
```

## 钩子模式

使用钩子进行验证和副作用：

```go
type User struct {
    ID       uint   `gorm:"primaryKey"`
    Email    string `gorm:"not null"`
    Password string `gorm:"not null"`
    UUID     string `gorm:"uniqueIndex"`
}

// 创建前钩子
func (u *User) BeforeCreate(tx *gorm.DB) error {
    if u.UUID == "" {
        u.UUID = uuid.New().String()
    }

    // 验证邮箱
    if !isValidEmail(u.Email) {
        return errors.New("邮箱格式无效")
    }

    return nil
}

// 更新前钩子
func (u *User) BeforeUpdate(tx *gorm.DB) error {
    // 防止邮箱变更
    if tx.Statement.Changed("Email") {
        return errors.New("邮箱不能更改")
    }
    return nil
}

// 创建后钩子
func (u *User) AfterCreate(tx *gorm.DB) error {
    // 发送欢迎邮件（异步）
    go sendWelcomeEmail(u.Email)
    return nil
}
```

## Scope 模式

使用 scopes 实现可复用查询逻辑：

```go
// 定义 scopes
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

// 使用 scopes
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

## 批量操作

使用批量操作提高效率：

```go
// 批量插入
users := []User{
    {Name: "用户 1", Email: "user1@example.com"},
    {Name: "用户 2", Email: "user2@example.com"},
}
db.CreateInBatches(users, 100) // 每批插入 100 条记录

// 批量更新
db.Model(&User{}).
    Where("status = ?", "pending").
    Update("status", "processed")

// 批量删除
db.Where("created_at < ?", time.Now().AddDate(-1, 0, 0)).
    Delete(&User{})
```

## 乐观锁

实现乐观锁处理并发更新：

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
        return ErrConcurrentUpdate // 乐观锁失败
    }

    return nil
}
```

## 参考

详细的 Go 模式请参阅 skill: `golang-patterns`。
高级 GORM 模式请参阅 skill: `gorm-best-practices`。
