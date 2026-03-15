---
paths:
  - "**/*.go"
  - "**/go.mod"
---
# Gin Hooks

> This file extends [common/hooks.md](../common/hooks.md) with Gin framework-specific content.

## PostToolUse Hooks

Configure in `~/.claude/settings.json`:

### Format with gofmt

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

### Run go vet

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

### Check Gin handler signatures

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": { "toolName": "Edit|Write", "filePath": "**/handler/**/*.go" },
        "hooks": [
          {
            "type": "command",
            "command": "grep -n 'func.*http.ResponseWriter.*http.Request' \"$FILE_PATH\" && echo 'WARNING: Use gin.Context instead of http.ResponseWriter' || true"
          }
        ]
      }
    ]
  }
}
```

### Verify build

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

### Run tests

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

### Check for common issues

```json
{
  "hooks": {
    "PreCommit": [
      {
        "type": "command",
        "command": "grep -r 'c.Bind(' --include='*.go' . && echo 'WARNING: Use ShouldBind instead of Bind' && exit 1 || true"
      },
      {
        "type": "command",
        "command": "grep -r 'context.Background()' --include='*.go' handler/ && echo 'WARNING: Use c.Request.Context() in handlers' && exit 1 || true"
      }
    ]
  }
}
```

### Security checks

```json
{
  "hooks": {
    "PreCommit": [
      {
        "type": "command",
        "command": "govulncheck ./... 2>/dev/null || echo 'govulncheck not installed'"
      }
    ]
  }
}
```

## Stop Hooks

### Check for debug code

```json
{
  "hooks": {
    "Stop": [
      {
        "type": "command",
        "command": "grep -r 'fmt.Println\\|log.Println' --include='*.go' handler/ && echo 'WARNING: Debug print statements found' || true"
      }
    ]
  }
}
```

### Check for TODO comments

```json
{
  "hooks": {
    "Stop": [
      {
        "type": "command",
        "command": "grep -r 'TODO\\|FIXME\\|HACK' --include='*.go' . && echo 'WARNING: Unresolved TODO comments' || true"
      }
    ]
  }
}
```

## Complete Gin Hooks Configuration

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
            "command": "grep -n 'c.Bind(' \"$FILE_PATH\" && echo 'WARNING: Consider using ShouldBind for better error handling' || true"
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
        "command": "grep -r 'fmt.Println' --include='*.go' handler/ && echo 'WARNING: Debug statements in handlers' || true"
      }
    ]
  }
}
```
