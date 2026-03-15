---
paths:
  - "**/*.go"
  - "**/go.mod"
---
# GORM Hooks

> This file extends [common/hooks.md](../common/hooks.md) with GORM-specific content.

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

### Check for SQL injection patterns

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": { "toolName": "Edit|Write", "filePath": "**/repository/**/*.go" },
        "hooks": [
          {
            "type": "command",
            "command": "grep -n 'fmt.Sprintf.*SELECT\\|fmt.Sprintf.*INSERT\\|fmt.Sprintf.*UPDATE\\|fmt.Sprintf.*DELETE' \"$FILE_PATH\" && echo 'WARNING: Potential SQL injection - use parameterized queries' || true"
          }
        ]
      }
    ]
  }
}
```

### Check for missing context

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": { "toolName": "Edit|Write", "filePath": "**/repository/**/*.go" },
        "hooks": [
          {
            "type": "command",
            "command": "grep -n 'db\\.' \"$FILE_PATH\" | grep -v 'WithContext' | grep -v '//' && echo 'WARNING: Consider using WithContext for context propagation' || true"
          }
        ]
      }
    ]
  }
}
```

### Verify model tags

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

### Run tests

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

### Check for common issues

```json
{
  "hooks": {
    "PreCommit": [
      {
        "type": "command",
        "command": "grep -r 'db.First.*\\)' --include='*.go' . | grep -v 'if err' | grep -v 'Error()' && echo 'WARNING: Unchecked GORM error' && exit 1 || true"
      },
      {
        "type": "command",
        "command": "grep -r 'Find(&.*)' --include='*.go' . | grep -v 'Preload' | grep -v 'N+1' && echo 'INFO: Check for N+1 queries - consider using Preload' || true"
      }
    ]
  }
}
```

### Database migration check

```json
{
  "hooks": {
    "PreCommit": [
      {
        "type": "command",
        "command": "if [ -d \"migrations\" ]; then ls -la migrations/; echo 'Remember to run migrations'; fi"
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
        "command": "grep -r 'fmt.Println\\|log.Println' --include='*.go' repository/ && echo 'WARNING: Debug print statements in repository' || true"
      }
    ]
  }
}
```

### Check for hardcoded credentials

```json
{
  "hooks": {
    "Stop": [
      {
        "type": "command",
        "command": "grep -r 'password.*=.*\"\\|secret.*=.*\"\\|api_key.*=.*\"' --include='*.go' . | grep -v 'os.Getenv' | grep -v 'viper' && echo 'WARNING: Possible hardcoded credentials' || true"
      }
    ]
  }
}
```

## Complete GORM Hooks Configuration

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
            "command": "grep -n 'fmt.Sprintf.*SELECT\\|fmt.Sprintf.*INSERT\\|fmt.Sprintf.*UPDATE' \"$FILE_PATH\" && echo 'WARNING: Use parameterized queries' || true"
          },
          {
            "type": "command",
            "command": "grep -n '\\.db\\.' \"$FILE_PATH\" | grep -v 'WithContext' | head -5 && echo 'INFO: Consider using WithContext' || true"
          }
        ]
      },
      {
        "matcher": { "toolName": "Edit|Write", "filePath": "**/model/**/*.go" },
        "hooks": [
          {
            "type": "command",
            "command": "grep -n 'json:"-"' \"$FILE_PATH\" || echo 'INFO: Remember to use json:\"-\" for sensitive fields'"
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
        "command": "grep -r 'fmt.Println' --include='*.go' repository/ model/ && echo 'WARNING: Debug statements found' || true"
      }
    ]
  }
}
```
