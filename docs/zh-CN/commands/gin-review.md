---
description: 全面的 Gin 框架代码审查，涵盖 RESTful API 设计、中间件模式、路由约定和安全性。调用 gin-reviewer 代理。
---

# Gin 代码审查

此命令调用 **gin-reviewer** 代理进行全面的 Gin 专用代码审查。

## 命令功能

1. **识别 Gin 更改**：通过 `git diff` 查找修改的处理器/路由文件
2. **运行静态分析**：执行 `go vet`、`staticcheck` 和代码检查器
3. **安全扫描**：检查 SQL 注入、CORS、认证问题
4. **API 审查**：分析 RESTful 约定、错误处理
5. **中间件审查**：检查中间件模式和顺序
6. **生成报告**：按严重程度分类问题

## 使用场景

在以下情况使用 `/gin-review`：
* 编写或修改 Gin 处理器后
* 提交 Gin 路由更改前
* 审查 API 端点
* 添加新中间件
* 接手 Gin 代码库

## 审查分类

### 关键（必须修复）
* 受保护路由缺少认证
* SQL 注入漏洞
* CORS 配置错误（通配符来源）
* 用户输入缺少验证
* 日志中包含敏感数据（密码、令牌）
* 公共端点缺少速率限制

### 高（应该修复）
* 使用 `Bind` 而非 `ShouldBind`
* 错误条件缺少错误响应
* 处理器超过 50 行（业务逻辑）
* 中间件缺少 `c.Next()`
* 错误的 HTTP 状态码

### 中（考虑修复）
* RESTful 命名不一致
* 缺少请求 ID 用于追踪
* 使用 `context.Background()` 而非 `c.Request.Context()`
* 路由定义重复
* 列表端点缺少分页

## 自动检查

```bash
# 静态分析
go vet ./...

# 高级检查（如已安装）
staticcheck ./...
golangci-lint run

# 竞态检测
go build -race ./...

# 安全漏洞
govulncheck ./...
```

## 示例用法

```text
用户: /gin-review

代理:
# Gin 代码审查报告

## 审查的文件
- internal/handler/user.go (已修改)
- internal/handler/auth.go (已修改)
- internal/middleware/auth.go (已修改)
- internal/routes/routes.go (已修改)

## 静态分析结果
✓ go vet: 无问题
✓ staticcheck: 无问题

## 发现的问题

[关键] 缺少输入验证
文件: internal/handler/user.go:28
问题: 邮箱字段未验证
```go
type CreateUserRequest struct {
    Email    string `json:"email"` // 缺少验证!
    Password string `json:"password"`
}
```
修复: 添加绑定验证
```go
type CreateUserRequest struct {
    Email    string `json:"email" binding:"required,email"`
    Password string `json:"password" binding:"required,min=8"`
}
```

[关键] CORS 配置错误
文件: internal/routes/routes.go:15
问题: AllowOrigins 设置为通配符
```go
config.AllowOrigins = []string{"*"} // 安全风险!
```
修复: 指定允许的来源
```go
config.AllowOrigins = []string{"https://example.com", "https://api.example.com"}
```

[高] 使用 Bind 而非 ShouldBind
文件: internal/handler/auth.go:42
问题: Bind 自动发送响应，无法自定义错误
```go
if err := c.Bind(&req); err != nil {
    return // 自动 400 响应
}
```
修复: 使用 ShouldBind 自定义错误处理
```go
if err := c.ShouldBindJSON(&req); err != nil {
    c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
    return
}
```

[高] 处理器臃肿
文件: internal/handler/order.go:156
问题: 处理器 87 行，包含业务逻辑
修复: 将业务逻辑提取到服务层

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

## Gin 专用检查

### 处理器质量
- [ ] 处理器少于 50 行
- [ ] 业务逻辑提取到服务层
- [ ] 使用 `ShouldBind` 进行验证
- [ ] 正确的错误响应和状态码

### 中间件质量
- [ ] 适当位置调用 `c.Next()`
- [ ] 错误响应前调用 `c.Abort()`
- [ ] 正确的中间件顺序（CORS → 认证 → 日志）
- [ ] 正确设置/获取上下文值

### 路由质量
- [ ] RESTful 命名约定
- [ ] 共享中间件使用路由组
- [ ] 无重复路由
- [ ] 一致的路径参数

### 安全性
- [ ] 受保护路由有认证
- [ ] 使用绑定标签进行输入验证
- [ ] CORS 正确配置
- [ ] 公共端点有速率限制
- [ ] 日志中无敏感数据

## 与其他命令的集成

* 先使用 `/gin-test` 确保处理器测试通过
* 如果出现构建错误使用 `/gin-build`
* 提交前使用 `/gin-review`
* 通用 Go 问题使用 `/go-review`

## 相关

* 代理: `agents/gin-reviewer.md`
* 技能: `skills/golang-patterns/`, `skills/api-design/`
