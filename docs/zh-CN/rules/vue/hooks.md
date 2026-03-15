---
paths:
  - "**/*.vue"
  - "**/*.ts"
  - "**/*.tsx"
---
# Vue Hooks

> 此文件扩展了 [common/hooks.md](../common/hooks.md) 的 Vue 特定内容。

## PostToolUse Hooks

在 `~/.claude/settings.json` 中配置：

### 带 Vue 规则的 ESLint

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": { "toolName": "Edit|Write" },
        "hooks": [
          {
            "type": "command",
            "command": "npx eslint --fix \"$FILE_PATH\" --ext .vue,.ts,.tsx"
          }
        ]
      }
    ]
  }
}
```

### Vue SFC 类型检查

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": { "toolName": "Edit|Write", "filePath": "**/*.vue" },
        "hooks": [
          {
            "type": "command",
            "command": "vue-tsc --noEmit"
          }
        ]
      }
    ]
  }
}
```

### Prettier 格式化

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": { "toolName": "Edit|Write" },
        "hooks": [
          {
            "type": "command",
            "command": "npx prettier --write \"$FILE_PATH\""
          }
        ]
      }
    ]
  }
}
```

## Stop Hooks

### console.log 审计

会话结束前检查所有修改的 Vue 文件中的 `console.log`：

```json
{
  "hooks": {
    "Stop": [
      {
        "type": "command",
        "command": "grep -r 'console\\.log' --include='*.vue' --include='*.ts' src/ && echo 'WARNING: console.log found in source files' || true"
      }
    ]
  }
}
```

### debugger 语句检查

```json
{
  "hooks": {
    "Stop": [
      {
        "type": "command",
        "command": "grep -r 'debugger' --include='*.vue' --include='*.ts' src/ && echo 'WARNING: debugger statements found' || true"
      }
    ]
  }
}
```

## PreCommit Hooks

### Vue 组件大小检查

```json
{
  "hooks": {
    "PreCommit": [
      {
        "type": "command",
        "command": "find src -name '*.vue' -exec wc -l {} \\; | awk '$1 > 500 { print $2 \" exceeds 500 lines\"; exit 1 }'"
      }
    ]
  }
}
```

### 类型检查

```json
{
  "hooks": {
    "PreCommit": [
      {
        "type": "command",
        "command": "vue-tsc --noEmit"
      }
    ]
  }
}
```

## 通知 Hooks

### 构建失败通知

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": { "toolName": "Bash", "output": ".*error.*" },
        "hooks": [
          {
            "type": "notification",
            "message": "Vue 构建失败，有错误"
          }
        ]
      }
    ]
  }
}
```

## Hook 配置模板

完整的 Vue hooks 配置：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": { "toolName": "Edit|Write" },
        "hooks": [
          {
            "type": "command",
            "command": "npx prettier --write \"$FILE_PATH\""
          },
          {
            "type": "command",
            "command": "npx eslint --fix \"$FILE_PATH\" --ext .vue,.ts,.tsx 2>/dev/null || true"
          }
        ]
      },
      {
        "matcher": { "toolName": "Edit|Write", "filePath": "**/*.vue" },
        "hooks": [
          {
            "type": "command",
            "command": "vue-tsc --noEmit 2>&1 | head -20 || true"
          }
        ]
      }
    ],
    "PreCommit": [
      {
        "type": "command",
        "command": "vue-tsc --noEmit"
      },
      {
        "type": "command",
        "command": "npm run lint"
      }
    ],
    "Stop": [
      {
        "type": "command",
        "command": "grep -r 'console\\.log\\|debugger' --include='*.vue' --include='*.ts' src/ && echo 'WARNING: Debug code found' || true"
      }
    ]
  }
}
```

## 参考

Vue 开发工作流程和自动化模式请参阅 skill: `vue-patterns`。
