---
paths:
  - "**/*.go"
  - "**/*_test.go"
---
# GORM 测试

> 此文件扩展了 [common/testing.md](../common/testing.md) 的 GORM 特定内容。

## 测试数据库设置

单元测试使用内存 SQLite：

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
        t.Fatalf("连接测试数据库失败: %v", err)
    }

    // 运行迁移
    err = db.AutoMigrate(&User{}, &Order{}, &OrderItem{})
    if err != nil {
        t.Fatalf("迁移失败: %v", err)
    }

    t.Cleanup(func() {
        sqlDB, _ := db.DB()
        sqlDB.Close()
    })

    return db
}
```

## 仓库测试

```go
func TestUserRepository_Create(t *testing.T) {
    db := setupTestDB(t)
    repo := NewUserRepository(db)
    ctx := context.Background()

    user := &User{
        Email:    "test@example.com",
        Name:     "测试用户",
        Password: "hashedpassword",
    }

    err := repo.Create(ctx, user)
    assert.NoError(t, err)
    assert.NotZero(t, user.ID)

    // 验证创建
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

## 表驱动测试

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
            name: "空列表",
            seedUsers: []User{},
            params: ListParams{Page: 1, PageSize: 10},
            wantCount: 0,
            wantTotal: 0,
        },
        {
            name: "单页",
            seedUsers: []User{
                {Email: "user1@example.com", Name: "用户 1"},
                {Email: "user2@example.com", Name: "用户 2"},
            },
            params: ListParams{Page: 1, PageSize: 10},
            wantCount: 2,
            wantTotal: 2,
        },
        {
            name: "分页",
            seedUsers: []User{
                {Email: "user1@example.com", Name: "用户 1"},
                {Email: "user2@example.com", Name: "用户 2"},
                {Email: "user3@example.com", Name: "用户 3"},
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

            // 准备数据
            for _, u := range tt.seedUsers {
                repo.Create(ctx, &u)
            }

            // 执行
            users, total, err := repo.List(ctx, tt.params)

            // 断言
            assert.NoError(t, err)
            assert.Len(t, users, tt.wantCount)
            assert.Equal(t, tt.wantTotal, total)
        })
    }
}
```

## 事务测试

```go
func TestOrderService_CreateOrder_Transaction(t *testing.T) {
    db := setupTestDB(t)

    // 准备库存有限的产品
    product := &Product{Name: "测试产品", Stock: 10}
    db.Create(product)

    svc := NewOrderService(db)
    ctx := context.Background()

    t.Run("成功订单", func(t *testing.T) {
        order := &Order{UserID: 1}
        items := []OrderItem{
            {ProductID: product.ID, Quantity: 5},
        }

        err := svc.CreateOrder(ctx, order, items)
        assert.NoError(t, err)

        // 验证库存扣减
        var updated Product
        db.First(&updated, product.ID)
        assert.Equal(t, 5, updated.Stock)
    })

    t.Run("库存不足回滚", func(t *testing.T) {
        order := &Order{UserID: 1}
        items := []OrderItem{
            {ProductID: product.ID, Quantity: 100}, // 超过可用库存
        }

        err := svc.CreateOrder(ctx, order, items)
        assert.Error(t, err)

        // 验证没有创建订单
        var count int64
        db.Model(&Order{}).Count(&count)
        assert.Equal(t, int64(0), count)
    })
}
```

## 关联测试

```go
func TestUserRepository_GetWithOrders(t *testing.T) {
    db := setupTestDB(t)
    repo := NewUserRepository(db)
    ctx := context.Background()

    // 创建带订单的用户
    user := &User{
        Email: "test@example.com",
        Name:  "测试用户",
        Orders: []Order{
            {TotalAmount: 100.00},
            {TotalAmount: 200.00},
        },
    }

    err := repo.Create(ctx, user)
    require.NoError(t, err)

    // 测试 Preload
    found, err := repo.GetWithOrders(ctx, user.ID)
    require.NoError(t, err)

    assert.Len(t, found.Orders, 2)
    assert.Equal(t, 100.00, found.Orders[0].TotalAmount)
}
```

## Mock 仓库

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
    return errors.New("未实现")
}

func (m *MockUserRepository) GetByID(ctx context.Context, id uint) (*User, error) {
    if m.GetByIDFunc != nil {
        return m.GetByIDFunc(ctx, id)
    }
    return nil, errors.New("未实现")
}

// ... 其他方法
```

## 测试辅助函数

```go
func seedTestData(t *testing.T, db *gorm.DB, models ...interface{}) {
    t.Helper()
    for _, model := range models {
        if err := db.Create(model).Error; err != nil {
            t.Fatalf("准备测试数据失败: %v", err)
        }
    }
}

func assertRecordExists(t *testing.T, db *gorm.DB, model interface{}, conditions ...interface{}) {
    t.Helper()
    result := db.First(model, conditions...)
    assert.NoError(t, result.Error, "期望记录存在")
}

func assertRecordNotExists(t *testing.T, db *gorm.DB, model interface{}, conditions ...interface{}) {
    t.Helper()
    result := db.First(model, conditions...)
    assert.ErrorIs(t, result.Error, gorm.ErrRecordNotFound, "期望记录不存在")
}

func countRecords(t *testing.T, db *gorm.DB, model interface{}) int64 {
    t.Helper()
    var count int64
    db.Model(model).Count(&count)
    return count
}
```

## 覆盖率目标

| 组件 | 目标 |
|-----------|--------|
| 仓库 CRUD | 95%+ |
| 自定义查询 | 90%+ |
| 事务 | 100% |
| 错误处理 | 100% |

```bash
# 带覆盖率运行测试
go test -cover ./repository/...

# 生成覆盖率报告
go test -coverprofile=coverage.out ./repository/...
go tool cover -html=coverage.out
```
