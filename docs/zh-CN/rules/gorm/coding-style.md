---
paths:
  - "**/*.go"
  - "**/go.mod"
---
# GORM 编码风格

> 此文件扩展了 [common/coding-style.md](../common/coding-style.md) 的 GORM 特定内容。

## 模型定义

使用完整的 GORM 标签并遵循命名约定：

```go
// 正确：带有正确标签的完整模型
type User struct {
    ID        uint           `gorm:"primaryKey" json:"id"`
    Email     string         `gorm:"size:255;uniqueIndex;not null" json:"email"`
    Name      string         `gorm:"size:100;not null" json:"name"`
    Password  string         `gorm:"size:255;not null" json:"-"` // 永不暴露敏感字段
    Status    string         `gorm:"size:20;default:'active';index" json:"status"`
    CreatedAt time.Time      `json:"createdAt"`
    UpdatedAt time.Time      `json:"updatedAt"`
    DeletedAt gorm.DeletedAt `gorm:"index" json:"-"`
}

// 错误：缺少标签和约束
type User struct {
    ID       int
    Email    string
    Name     string
    Password string
}
```

## 表命名

实现 `TableName()` 显式指定表名：

```go
// 正确：显式表名
func (User) TableName() string {
    return "users"
}

// 正确：带模式
func (User) TableName() string {
    return "public.users"
}

// 避免依赖默认复数形式
```

## 字段类型

使用适合数据库兼容性的字段类型：

```go
type Order struct {
    ID          uint          `gorm:"primaryKey"`
    TotalAmount float64       `gorm:"type:decimal(10,2);not null"` // 金额使用 decimal
    Quantity    int           `gorm:"not null;default:0"`
    Description string        `gorm:"type:text"` // 长字符串使用 text
    Status      string        `gorm:"size:20;not null;default:'pending'"`
    Metadata    datatypes.JSON `gorm:"type:jsonb"` // PostgreSQL JSON
    CreatedAt   time.Time
}
```

## 关联定义

使用显式外键定义关联：

```go
// 正确：显式外键和约束
type User struct {
    ID      uint     `gorm:"primaryKey"`
    Profile Profile  `gorm:"foreignKey:UserID;constraint:OnDelete:CASCADE"`
    Orders  []Order  `gorm:"foreignKey:UserID;constraint:OnDelete:SET NULL"`
    Roles   []Role   `gorm:"many2many:user_roles;"`
}

type Order struct {
    ID          uint        `gorm:"primaryKey"`
    UserID      *uint       `gorm:"index"` // 指针表示可空外键
    User        User        `gorm:"foreignKey:UserID"`
    OrderItems  []OrderItem `gorm:"foreignKey:OrderID"`
}

// 错误：缺少约束
type User struct {
    ID      uint
    Orders  []Order // 没有外键定义
}
```

## 仓库模式

使用仓库模式进行数据访问：

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

## 上下文传播

始终使用 `WithContext`：

```go
// 正确：传播上下文
func (r *userRepository) GetByID(ctx context.Context, id uint) (*User, error) {
    var user User
    err := r.db.WithContext(ctx).First(&user, id).Error
    return &user, err
}

// 错误：没有上下文
func (r *userRepository) GetByID(id uint) (*User, error) {
    var user User
    err := r.db.First(&user, id).Error
    return &user, err
}
```

## 错误处理

正确处理 GORM 错误：

```go
func (r *userRepository) GetByID(ctx context.Context, id uint) (*User, error) {
    var user User
    err := r.db.WithContext(ctx).First(&user, id).Error
    if err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, ErrUserNotFound // 自定义错误
        }
        return nil, fmt.Errorf("获取用户失败: %w", err)
    }
    return &user, nil
}
```

## 查询构建

链式方法构建可读查询：

```go
// 正确：清晰的查询链
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
