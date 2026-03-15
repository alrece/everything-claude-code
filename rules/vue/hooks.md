---
paths:
  - "**/*.vue"
  - "**/*.ts"
  - "**/*.tsx"
---
# Vue Hooks

> This file extends [common/hooks.md](../common/hooks.md) with Vue specific content.

## PostToolUse Hooks

Configure in `~/.claude/settings.json`:

### ESLint with Vue Rules

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

### Vue SFC Type Check

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

### Prettier Format

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

### console.log Audit

Check all modified Vue files for `console.log` before session ends:

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

### Debugger Statement Check

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

### Vue Component Size Check

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

### Type Check

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

## Notification Hooks

### Build Failure Notification

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": { "toolName": "Bash", "output": ".*error.*" },
        "hooks": [
          {
            "type": "notification",
            "message": "Vue build failed with errors"
          }
        ]
      }
    ]
  }
}
```

## Hook Configuration Template

Complete Vue hooks configuration:

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

## Reference

See skill: `vue-patterns` for Vue development workflow and automation patterns.
