---
description: GORM 仓库的 TDD 工作流。先使用 mock 或测试容器编写仓库测试，再实现。确保 80%+ 覆盖率。
---

# GORM TDD 命令

此命令为 GORM 仓库和数据访问层强制执行测试驱动开发方法论。

## 命令功能

1. **定义仓库接口**：首先搭建仓库方法
2. **编写仓库测试**：创建全面的测试用例（RED）
3. **运行测试**：验证测试因正确原因失败
4. **实现仓库**：编写最小代码通过测试（GREEN）
5. **重构**：改进代码，保持测试通过
6. **检查覆盖率**：确保 80%+ 仓库覆盖

## 使用场景

在以下情况使用 `/gorm-test`：
* 实现新的 GORM 仓库
* 添加数据库查询方法
* 测试模型关联
* 修复仓库 bug（先写失败测试）
* 构建关键数据访问逻辑

## TDD 循环

```
RED     → 编写失败的仓库测试
GREEN   → 实现最小仓库方法
REFACTOR → 优化查询，测试保持通过
REPEAT  → 下一个测试用例
```

## 示例会话

````
用户: /gorm-test 我需要一个根据邮箱查找用户的仓库

代理:
# TDD 会话: UserRepository.FindByEmail

## 步骤 1: 定义仓库接口

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
    ErrUserNotFound = errors.New("用户未找到")
)

type UserRepository struct {
    db *gorm.DB
}

func NewUserRepository(db *gorm.DB) *UserRepository {
    return &UserRepository{db: db}
}

// FindByEmail 根据邮箱地址查找用户
func (r *UserRepository) FindByEmail(ctx context.Context, email string) (*model.User, error) {
    panic("未实现")
}
```

## 步骤 2: 编写测试 (RED)

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
            name: "找到用户",
            seedUsers: []*model.User{
                {Email: "test@example.com", Name: "测试用户"},
            },
            email:    "test@example.com",
            wantUser: &model.User{Email: "test@example.com", Name: "测试用户"},
            wantErr:  nil,
        },
        {
            name:      "用户未找到",
            seedUsers: []*model.User{},
            email:     "nonexistent@example.com",
            wantUser:  nil,
            wantErr:   ErrUserNotFound,
        },
        {
            name: "邮箱不区分大小写",
            seedUsers: []*model.User{
                {Email: "Test@Example.COM", Name: "测试用户"},
            },
            email:    "test@example.com",
            wantUser: &model.User{Email: "Test@Example.COM", Name: "测试用户"},
            wantErr:  nil,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            db := setupTestDB(t)
            repo := NewUserRepository(db)

            // 准备数据
            for _, u := range tt.seedUsers {
                err := db.Create(u).Error
                require.NoError(t, err)
            }

            // 执行
            got, err := repo.FindByEmail(ctx, tt.email)

            // 断言
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

## 步骤 3: 运行测试 - 验证失败

```bash
$ go test ./repository/...

--- FAIL: TestUserRepository_FindByEmail (0.00s)
    panic: 未实现
FAIL
```

✓ 测试按预期失败。

## 步骤 4: 实现仓库 (GREEN)

```go
// repository/user.go
package repository

import (
    "context"
    "errors"

    "gorm.io/gorm"
    "project/internal/model"
)

var ErrUserNotFound = errors.New("用户未找到")

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

## 步骤 5: 运行测试 - 验证通过

```bash
$ go test ./repository/...

PASS
ok      project/internal/repository   0.025s
```

✓ 所有测试通过！

## 步骤 6: 检查覆盖率

```bash
$ go test -cover ./repository/...

PASS
coverage: 95.0% of statements
ok      project/internal/repository   0.025s
```

✓ 覆盖率: 95%

## TDD 完成！
````

## 测试模式

### 测试数据库设置
```go
// 单元测试使用 SQLite 内存数据库
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

// 集成测试使用 testcontainers
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

### 仓库接口测试
```go
type UserRepository interface {
    FindByID(ctx context.Context, id uint) (*model.User, error)
    FindByEmail(ctx context.Context, email string) (*model.User, error)
    Create(ctx context.Context, user *model.User) error
    Update(ctx context.Context, user *model.User) error
    Delete(ctx context.Context, id uint) error
}
```

### 关联测试
```go
func TestUserOrdersAssociation(t *testing.T) {
    db := setupTestDB(t)
    repo := NewUserRepository(db)

    // 创建带订单的用户
    user := &model.User{
        Email: "test@example.com",
        Orders: []model.Order{
            {Total: 100.00},
            {Total: 200.00},
        },
    }

    err := repo.Create(context.Background(), user)
    require.NoError(t, err)

    // 测试 Preload
    got, err := repo.FindByID(context.Background(), user.ID)
    require.NoError(t, err)
    assert.Len(t, got.Orders, 2)
}
```

### 事务测试
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

    // 验证订单已创建
    var count int64
    db.Model(&model.Order{}).Count(&count)
    assert.Equal(t, int64(1), count)

    // 验证项目已创建
    db.Model(&model.OrderItem{}).Count(&count)
    assert.Equal(t, int64(3), count)
}
```

## 覆盖率命令

```bash
# 基本覆盖率
go test -cover ./repository/...

# 覆盖率配置文件
go test -coverprofile=coverage.out ./repository/...

# 在浏览器中查看
go tool cover -html=coverage.out

# 按函数覆盖率
go tool cover -func=coverage.out
```

## 覆盖率目标

| 仓库类型 | 目标 |
|-----------------|--------|
| 关键数据访问（认证、支付） | 100% |
| 公共仓库 | 90%+ |
| 通用仓库 | 80%+ |
| 只读查询 | 70%+ |

## TDD 最佳实践

**应该做：**
* 在任何实现之前先写测试
* 单元测试使用内存 SQLite 以提高速度
* 集成测试使用 testcontainers
* 使用 Preload 测试关联
* 测试错误条件（未找到、约束错误）

**不应该做：**
* 使用生产数据库进行测试
* 跳过错误路径测试
* 忽略事务回滚场景
* 测试 GORM 内部（测试你的逻辑）
* 在测试中使用 `time.Sleep`

## 相关命令

* `/gorm-build` - 修复构建错误
* `/gorm-review` - 实现后审查代码
* `/go-test` - 通用 Go 测试

## 相关

* 技能: `skills/golang-testing/`
* 代理: `agents/gorm-reviewer.md`
