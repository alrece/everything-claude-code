---
description: 增量修复 Gin 框架构建错误、路由问题、中间件问题和绑定错误。调用 gin-build-resolver 代理。
---

# Gin 构建和修复

此命令调用 **gin-build-resolver** 代理，以最小化改动增量修复 Gin 框架错误。

## 命令功能

1. **运行诊断**：执行 `go build`，检查路由冲突、中间件链
2. **解析错误**：按处理器/中间件分组并按严重程度排序
3. **增量修复**：一次修复一个错误
4. **验证修复**：每次更改后重新构建
5. **报告摘要**：显示已修复和剩余的问题

## 使用场景

在以下情况使用 `/gin-build`：
* Gin 路由错误（重复路由、冲突）
* 中间件链问题（缺少 `c.Next()`、顺序错误）
* 绑定/验证错误（`Bind` vs `ShouldBind`）
* 上下文使用错误（空指针、goroutine 泄漏）
* 处理器签名不匹配

## 诊断命令

```bash
# 主要构建检查
go build ./...

# 静态分析
go vet ./...

# 检查 Gin 特定问题
staticcheck ./...

# 运行测试
go test ./...
```

## 示例会话

```text
用户: /gin-build

代理:
# Gin 构建问题解决

## 初始诊断

```bash
$ go build ./...
internal/handler/user.go:25:15: cannot use handler (type func(http.ResponseWriter, *http.Request)) as type gin.HandlerFunc
internal/handler/user.go:42:9: undefined: c.ShouldBindJSON
internal/middleware/auth.go:18:2: c.Next declared but not called
```

发现错误: 3

## 修复 1: 处理器签名不匹配

文件: internal/handler/user.go:25
错误: cannot use handler as type gin.HandlerFunc
原因: 使用 net/http 签名而非 gin.Context

```go
// 从
func GetUser(w http.ResponseWriter, r *http.Request) {
    // ...
}

// 改为
func GetUser(c *gin.Context) {
    // ...
}
```

```bash
$ go build ./...
# 剩余 2 个错误
```

## 修复 2: 未定义方法

文件: internal/handler/user.go:42
错误: undefined: c.ShouldBindJSON
原因: 缺少 gin 导入或拼写错误

```go
// 添加导入
import "github.com/gin-gonic/gin"

// 修复绑定调用
var req CreateUserRequest
if err := c.ShouldBindJSON(&req); err != nil {
    c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
    return
}
```

```bash
$ go build ./...
# 剩余 1 个错误
```

## 修复 3: 中间件缺少 Next()

文件: internal/middleware/auth.go:18
错误: c.Next declared but not called

```go
func AuthMiddleware(c *gin.Context) {
    token := c.GetHeader("Authorization")
    if token == "" {
        c.AbortWithStatusJSON(401, gin.H{"error": "unauthorized"})
        return
    }
    // 验证令牌...
    c.Next() // 添加了这一行
}
```

```bash
$ go build ./...
# 构建成功!
```

## 最终验证

```bash
$ go test ./...
ok      project/internal/handler   0.023s
ok      project/internal/middleware   0.015s
```

## 摘要

| 指标 | 数量 |
|--------|-------|
| 修复的构建错误 | 3 |
| 修改的文件 | 2 |
| 剩余问题 | 0 |

构建状态: ✅ 成功
```

## 常见错误修复

| 错误 | 典型修复方法 |
|-------|-------------|
| `cannot use handler as gin.HandlerFunc` | 将签名改为 `func(c *gin.Context)` |
| `route conflicts with existing route` | 修改路径或使用路由组 |
| `undefined: c.ShouldBind` | 导入 `github.com/gin-gonic/gin` |
| `c.Next declared but not called` | 在中间件中添加 `c.Next()` |
| `nil pointer dereference` | 检查依赖注入 |
| `context canceled` | 使用 `c.Request.Context()` |

## 修复策略

1. **处理器签名** - 确保所有处理器使用 `func(c *gin.Context)`
2. **路由冲突** - 检查重复路由
3. **中间件链** - 验证调用了 `c.Next()`
4. **绑定错误** - 使用 `ShouldBind` 而非 `Bind`
5. **上下文使用** - 使用 `c.Request.Context()` 进行传播

## 停止条件

代理将在以下情况停止并报告：
* 同一错误在 3 次尝试后仍然存在
* 修复引入更多错误
* 需要架构更改
* 缺少外部依赖

## 相关命令

* `/gin-review` - 审查 Gin 代码质量
* `/gin-test` - 测试 Gin 处理器
* `/go-build` - 通用 Go 构建问题

## 相关

* 代理: `agents/gin-build-resolver.md`
* 技能: `skills/golang-patterns/`
