---
description: 增量修复 GORM 模型定义、关联、迁移和查询问题。调用 gorm-build-resolver 代理进行最小化、精准修复。
---

# GORM 构建和修复

此命令调用 **gorm-build-resolver** 代理，以最小化改动增量修复 GORM 错误。

## 命令功能

1. **运行诊断**：执行 `go build`，检查模型定义，验证迁移
2. **解析错误**：按模型/仓库分组并按严重程度排序
3. **增量修复**：一次修复一个错误
4. **验证修复**：每次更改后重新构建和测试
5. **报告摘要**：显示已修复和剩余的问题

## 使用场景

在以下情况使用 `/gorm-build`：
* GORM 模型定义错误
* 关联/关系问题
* 迁移错误
* 查询语法错误
* 钩子/回调问题

## 诊断命令

```bash
# 主要构建检查
go build ./...

# 静态分析
go vet ./...

# 检查 GORM 特定问题
staticcheck ./...

# 详细运行测试
go test -v ./...
```

## 示例会话

```text
用户: /gorm-build

代理:
# GORM 构建问题解决

## 初始诊断

```bash
$ go build ./...
internal/model/user.go:25:15: undefined: gorm.DeletedAt
internal/model/order.go:42:9: cannot use Order as type *Order in association
internal/repository/user.go:58:2: undefined: db.Preload
```

发现错误: 3

## 修复 1: 缺少 GORM 导入

文件: internal/model/user.go:25
错误: undefined: gorm.DeletedAt
原因: 缺少 gorm 导入

```go
// 添加导入
import "gorm.io/gorm"

// 模型现在可编译
type User struct {
    ID        uint           `gorm:"primaryKey"`
    Email     string         `gorm:"uniqueIndex"`
    DeletedAt gorm.DeletedAt `gorm:"index"`
}
```

```bash
$ go build ./...
# 剩余 2 个错误
```

## 修复 2: 关联类型不匹配

文件: internal/model/order.go:42
错误: cannot use Order as type *Order in association
原因: 关联中指针/值不匹配

```go
// 从
type User struct {
    Orders []Order `gorm:"foreignKey:UserID"`
}

// 改为
type User struct {
    Orders []Order `gorm:"foreignKey:UserID;references:ID"`
}

// 同时修复 Order 模型
type Order struct {
    ID     uint  `gorm:"primaryKey"`
    UserID *uint `gorm:"index"` // 指针表示可空外键
    User   User  `gorm:"foreignKey:UserID"`
}
```

```bash
$ go build ./...
# 剩余 1 个错误
```

## 修复 3: 缺少 GORM 方法

文件: internal/repository/user.go:58
错误: undefined: db.Preload
原因: 缺少 gorm.DB 导入或 DB 类型错误

```go
// 修复导入
import "gorm.io/gorm"

// 修复仓库
func (r *UserRepository) GetUsersWithOrders(ctx context.Context) ([]User, error) {
    var users []User
    err := r.db.WithContext(ctx).
        Preload("Orders"). // 现在可用
        Find(&users).Error
    return users, err
}
```

```bash
$ go build ./...
# 构建成功!
```

## 最终验证

```bash
$ go test ./...
ok      project/internal/model       0.012s
ok      project/internal/repository  0.025s
```

## 摘要

| 指标 | 数量 |
|--------|-------|
| 修复的构建错误 | 3 |
| 修改的文件 | 3 |
| 剩余问题 | 0 |

构建状态: ✅ 成功
```

## 常见错误修复

| 错误 | 典型修复方法 |
|-------|-------------|
| `undefined: gorm.X` | 导入 `gorm.io/gorm` |
| `invalid field type` | 使用支持的类型或自定义扫描器 |
| `unknown column` | 添加 `gorm:"column:name"` 标签 |
| `table not found` | 实现 `TableName()` 方法 |
| `invalid association` | 添加 `foreignKey` 和 `references` 标签 |
| `record not found` in preload | 检查关联字段名称 |
| `constraint error` | 验证外键完整性 |
| `duplicate entry` | 添加唯一约束或插入前检查 |

## 修复策略

1. **模型定义** - 确保正确的 gorm 标签和类型
2. **关联** - 验证外键和引用
3. **迁移** - 检查 AutoMigrate 调用
4. **查询** - 修复方法链和语法
5. **钩子** - 确保正确的签名

## 停止条件

代理将在以下情况停止并报告：
* 同一错误在 3 次尝试后仍然存在
* 修复引入更多错误
* 需要超出范围的架构更改
* 缺少外部依赖

## 相关命令

* `/gorm-review` - 审查 GORM 代码质量
* `/gorm-test` - 测试 GORM 仓库
* `/go-build` - 通用 Go 构建问题

## 相关

* 代理: `agents/gorm-build-resolver.md`
* 技能: `skills/golang-patterns/`
