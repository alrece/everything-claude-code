---
paths:
  - "**/*.go"
  - "**/go.mod"
---
# Gin Hooks

> 此文件扩展了 [common/hooks.md](../common/hooks.md) 的 Gin 框架特定内容。

## PostToolUse Hooks

在 `~/.claude/settings.json` 中配置：

### 使用 gofmt 格式化

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": { "toolName": "Edit|Write", "filePath": "**/*.go" },
        "hooks": [
          {
            "type": "command",
            "command": "gofmt -w \"$FILE_PATH\""
          }
        ]
      }
    ]
  }
}
```

### 运行 go vet

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": { "toolName": "Edit|Write", "filePath": "**/*.go" },
        "hooks": [
          {
            "type": "command",
            "command": "go vet ./... 2>&1 | head -20 || true"
          }
        ]
      }
    ]
  }
}
```

### 检查 Gin 处理器签名

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": { "toolName": "Edit|Write", "filePath": "**/handler/**/*.go" },
        "hooks": [
          {
            "type": "command",
            "command": "grep -n 'func.*http.ResponseWriter.*http.Request' \"$FILE_PATH\" && echo 'WARNING: 使用 gin.Context 而非 http.ResponseWriter' || true"
          }
        ]
      }
    ]
  }
}
```

### 验证构建

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": { "toolName": "Edit|Write", "filePath": "**/*.go" },
        "hooks": [
          {
            "type": "command",
            "command": "go build ./... 2>&1 | head -20 || true"
          }
        ]
      }
    ]
  }
}
```

## PreCommit Hooks

### 运行测试

```json
{
  "hooks": {
    "PreCommit": [
      {
        "type": "command",
        "command": "go test ./... -short"
      }
    ]
  }
}
```

### 检查常见问题

```json
{
  "hooks": {
    "PreCommit": [
      {
        "type": "command",
        "command": "grep -r 'c.Bind(' --include='*.go' . && echo 'WARNING: 使用 ShouldBind 而非 Bind' && exit 1 || true"
      },
      {
        "type": "command",
        "command": "grep -r 'context.Background()' --include='*.go' handler/ && echo 'WARNING: 在处理器中使用 c.Request.Context()' && exit 1 || true"
      }
    ]
  }
}
```

### 安全检查

```json
{
  "hooks": {
    "PreCommit": [
      {
        "type": "command",
        "command": "govulncheck ./... 2>/dev/null || echo 'govulncheck 未安装'"
      }
    ]
  }
}
```

## Stop Hooks

### 检查调试代码

```json
{
  "hooks": {
    "Stop": [
      {
        "type": "command",
        "command": "grep -r 'fmt.Println\\|log.Println' --include='*.go' handler/ && echo 'WARNING: 处理器中有调试打印语句' || true"
      }
    ]
  }
}
```

### 检查 TODO 注释

```json
{
  "hooks": {
    "Stop": [
      {
        "type": "command",
        "command": "grep -r 'TODO\\|FIXME\\|HACK' --include='*.go' . && echo 'WARNING: 未解决的 TODO 注释' || true"
      }
    ]
  }
}
```

## 完整的 Gin Hooks 配置

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": { "toolName": "Edit|Write", "filePath": "**/*.go" },
        "hooks": [
          {
            "type": "command",
            "command": "gofmt -w \"$FILE_PATH\""
          },
          {
            "type": "command",
            "command": "go vet ./... 2>&1 | head -20 || true"
          }
        ]
      },
      {
        "matcher": { "toolName": "Edit|Write", "filePath": "**/handler/**/*.go" },
        "hooks": [
          {
            "type": "command",
            "command": "grep -n 'c.Bind(' \"$FILE_PATH\" && echo 'WARNING: 考虑使用 ShouldBind 以获得更好的错误处理' || true"
          }
        ]
      }
    ],
    "PreCommit": [
      {
        "type": "command",
        "command": "go test ./... -short"
      },
      {
        "type": "command",
        "command": "go build ./..."
      }
    ],
    "Stop": [
      {
        "type": "command",
        "command": "grep -r 'fmt.Println' --include='*.go' handler/ && echo 'WARNING: 处理器中有调试语句' || true"
      }
    ]
  }
}
```
