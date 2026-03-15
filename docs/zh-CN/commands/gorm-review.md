---
description: 全面的 GORM 代码审查，涵盖模型设计、关联、查询优化和数据库模式。调用 gorm-reviewer 代理。
---

# GORM 代码审查

此命令调用 **gorm-reviewer** 代理进行全面的 GORM 专用代码审查。

## 命令功能

1. **识别 GORM 更改**：通过 `git diff` 查找修改的模型/仓库文件
2. **运行静态分析**：执行 `go vet`、`staticcheck` 和代码检查器
3. **安全扫描**：检查 SQL 注入、敏感数据暴露
4. **模型审查**：分析模型定义、约束、索引
5. **查询审查**：检查 N+1 查询、缺少预加载
6. **生成报告**：按严重程度分类问题

## 使用场景

在以下情况使用 `/gorm-review`：
* 编写或修改 GORM 模型后
* 提交仓库更改前
* 审查数据库架构更改
* 添加新关联
* 接手 GORM 代码库

## 审查分类

### 关键（必须修复）
* 字符串拼接导致的 SQL 注入
* 多表操作缺少事务
* 敏感数据硬删除（无软删除）
* 密码/敏感字段未加密
* 关键字段缺少约束（not null、unique）

### 高（应该修复）
* N+1 查询（缺少 Preload）
* 频繁查询的列缺少索引
* 孤立记录（缺少外键约束）
* 竞态条件（无乐观锁）
* 缺少 CreatedAt/UpdatedAt 时间戳

### 中（考虑修复）
* 查询中缺少上下文
* 应使用 `First` 时使用 `Find`
* 未处理 `gorm.ErrRecordNotFound`
* 缺少软删除（DeletedAt）
* 低效查询模式

## 自动检查

```bash
# 静态分析
go vet ./...

# 高级检查（如已安装）
staticcheck ./...
golangci-lint run

# 安全漏洞
govulncheck ./...

# 带竞态检测测试
go test -race ./...
```

## 示例用法

```text
用户: /gorm-review

代理:
# GORM 代码审查报告

## 审查的文件
- internal/model/user.go (已修改)
- internal/model/order.go (已修改)
- internal/repository/user.go (已修改)

## 静态分析结果
✓ go vet: 无问题
✓ staticcheck: 无问题

## 发现的问题

[关键] SQL 注入风险
文件: internal/repository/user.go:45
问题: 原始 SQL 中字符串拼接
```go
db.Raw("SELECT * FROM users WHERE email = '" + email + "'").Scan(&user)
```
修复: 使用参数化查询
```go
db.Where("email = ?", email).First(&user)
// 或使用 Raw
db.Raw("SELECT * FROM users WHERE email = ?", email).Scan(&user)
```

[关键] 缺少事务
文件: internal/repository/order.go:78
问题: 多个数据库操作没有事务
```go
func CreateOrder(order *Order, items []OrderItem) error {
    db.Create(order)          // 操作 1
    for _, item := range items {
        db.Create(&item)       // 操作 2+
    }
    return nil // 如果操作 3 失败怎么办？
}
```
修复: 包装在事务中
```go
func CreateOrder(order *Order, items []OrderItem) error {
    return db.Transaction(func(tx *gorm.DB) error {
        if err := tx.Create(order).Error; err != nil {
            return err
        }
        for _, item := range items {
            item.OrderID = order.ID
            if err := tx.Create(&item).Error; err != nil {
                return err // 将回滚
            }
        }
        return nil
    })
}
```

[高] N+1 查询问题
文件: internal/repository/user.go:34
问题: Orders 关联缺少 Preload
```go
func GetUsers() ([]User, error) {
    var users []User
    db.Find(&users)
    for i := range users {
        db.Model(&users[i]).Association("Orders").Find(&users[i].Orders)
    }
    return users, nil
}
```
修复: 使用 Preload
```go
func GetUsers() ([]User, error) {
    var users []User
    err := db.Preload("Orders").Find(&users).Error
    return users, err
}
```

[高] 缺少索引
文件: internal/model/user.go:15
问题: Email 字段频繁查询但未索引
```go
type User struct {
    ID    uint   `gorm:"primaryKey"`
    Email string `json:"email"` // 缺少索引!
}
```
修复: 添加索引
```go
type User struct {
    ID    uint   `gorm:"primaryKey"`
    Email string `gorm:"uniqueIndex" json:"email"`
}
```

## 摘要
- 关键: 2
- 高: 2
- 中: 0

建议: ❌ 修复关键问题前阻止合并
```

## 批准标准

| 状态 | 条件 |
|--------|-----------|
| ✅ 批准 | 没有关键或高优先级问题 |
| ⚠️ 警告 | 仅存在中优先级问题（谨慎合并） |
| ❌ 阻止 | 发现关键或高优先级问题 |

## GORM 专用检查

### 模型质量
- [ ] 正确的 gorm 标签（primaryKey、uniqueIndex、not null）
- [ ] 存在 CreatedAt/UpdatedAt 字段
- [ ] DeletedAt 用于软删除（如适用）
- [ ] 实现 TableName() 用于显式命名
- [ ] 适当的字段类型兼容数据库

### 关联质量
- [ ] 正确的外键标签
- [ ] 适当的约束标签（OnDelete、OnUpdate）
- [ ] 定义多对多连接表
- [ ] 可空外键使用指针类型

### 查询质量
- [ ] 关联使用 Preload
- [ ] 使用 WithContext 传播上下文
- [ ] ErrRecordNotFound 错误处理
- [ ] 多操作变更使用事务
- [ ] 频繁查询列有索引

### 安全性
- [ ] 查询中无字符串拼接
- [ ] 敏感字段不记录日志
- [ ] 密码存储前加密
- [ ] 可恢复数据使用软删除

## 与其他命令的集成

* 先使用 `/gorm-test` 确保仓库测试通过
* 如果出现构建错误使用 `/gorm-build`
* 提交前使用 `/gorm-review`
* 通用 Go 问题使用 `/go-review`

## 相关

* 代理: `agents/gorm-reviewer.md`
* 技能: `skills/golang-patterns/`, `skills/database-patterns/`
