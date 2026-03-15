---
paths:
  - "**/*.go"
  - "**/go.mod"
---
# GORM Hooks

> 此文件扩展了 [common/hooks.md](../common/hooks.md) 的 GORM 特定内容。

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

### 检查 SQL 注入模式

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": { "toolName": "Edit|Write", "filePath": "**/repository/**/*.go" },
        "hooks": [
          {
            "type": "command",
            "command": "grep -n 'fmt.Sprintf.*SELECT\\|fmt.Sprintf.*INSERT\\|fmt.Sprintf.*UPDATE\\|fmt.Sprintf.*DELETE' \"$FILE_PATH\" && echo 'WARNING: 潜在 SQL 注入 - 使用参数化查询' || true"
          }
        ]
      }
    ]
  }
}
```

### 检查缺少上下文

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": { "toolName": "Edit|Write", "filePath": "**/repository/**/*.go" },
        "hooks": [
          {
            "type": "command",
            "command": "grep -n 'db\\.' \"$FILE_PATH\" | grep -v 'WithContext' | grep -v '//' && echo 'WARNING: 考虑使用 WithContext 进行上下文传播' || true"
          }
        ]
      }
    ]
  }
}
```

### 验证模型标签

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": { "toolName": "Edit|Write", "filePath": "**/model/**/*.go" },
        "hooks": [
          {
            "type": "command",
            "command": "grep -n 'type.*struct' -A 20 \"$FILE_PATH\" | grep -v 'gorm:' | grep -v 'json:' | head -5 || true"
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
        "command": "go test ./repository/... -short"
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
        "command": "grep -r 'db.First.*\\)' --include='*.go' . | grep -v 'if err' | grep -v 'Error()' && echo 'WARNING: 未检查 GORM 错误' && exit 1 || true"
      },
      {
        "type": "command",
        "command": "grep -r 'Find(&.*)' --include='*.go' . | grep -v 'Preload' | grep -v 'N+1' && echo 'INFO: 检查 N+1 查询 - 考虑使用 Preload' || true"
      }
    ]
  }
}
```

### 数据库迁移检查

```json
{
  "hooks": {
    "PreCommit": [
      {
        "type": "command",
        "command": "if [ -d \"migrations\" ]; then ls -la migrations/; echo '记得运行迁移'; fi"
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
        "command": "grep -r 'fmt.Println\\|log.Println' --include='*.go' repository/ && echo 'WARNING: 仓库中有调试打印语句' || true"
      }
    ]
  }
}
```

### 检查硬编码凭据

```json
{
  "hooks": {
    "Stop": [
      {
        "type": "command",
        "command": "grep -r 'password.*=.*\"\\|secret.*=.*\"\\|api_key.*=.*\"' --include='*.go' . | grep -v 'os.Getenv' | grep -v 'viper' && echo 'WARNING: 可能存在硬编码凭据' || true"
      }
    ]
  }
}
```

## 完整的 GORM Hooks 配置

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
      },
      {
        "matcher": { "toolName": "Edit|Write", "filePath": "**/repository/**/*.go" },
        "hooks": [
          {
            "type": "command",
            "command": "grep -n 'fmt.Sprintf.*SELECT\\|fmt.Sprintf.*INSERT\\|fmt.Sprintf.*UPDATE' \"$FILE_PATH\" && echo 'WARNING: 使用参数化查询' || true"
          },
          {
            "type": "command",
            "command": "grep -n '\\.db\\.' \"$FILE_PATH\" | grep -v 'WithContext' | head -5 && echo 'INFO: 考虑使用 WithContext' || true"
          }
        ]
      },
      {
        "matcher": { "toolName": "Edit|Write", "filePath": "**/model/**/*.go" },
        "hooks": [
          {
            "type": "command",
            "command": "grep -n 'json:"-"' \"$FILE_PATH\" || echo 'INFO: 记得为敏感字段使用 json:\"-\"'"
          }
        ]
      }
    ],
    "PreCommit": [
      {
        "type": "command",
        "command": "go test ./repository/... -short"
      },
      {
        "type": "command",
        "command": "go vet ./..."
      }
    ],
    "Stop": [
      {
        "type": "command",
        "command": "grep -r 'fmt.Println' --include='*.go' repository/ model/ && echo 'WARNING: 发现调试语句' || true"
      }
    ]
  }
}
```
