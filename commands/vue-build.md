---
description: Fix Vue build errors, SFC compilation issues, and Vite/Webpack config problems incrementally. Invokes the vue-build-resolver agent for minimal, surgical fixes.
---

# Vue Build and Fix

This command invokes the **vue-build-resolver** agent to incrementally fix Vue build errors with minimal changes.

## What This Command Does

1. **Run Diagnostics**: Execute `npm run build`, `vue-tsc --noEmit`
2. **Parse Errors**: Group by file and sort by severity
3. **Fix Incrementally**: One error at a time
4. **Verify Each Fix**: Re-run build after each change
5. **Report Summary**: Show what was fixed and what remains

## When to Use

Use `/vue-build` when:
- `npm run build` fails with errors
- Vue SFC compilation errors occur
- `vue-tsc` reports type errors
- Vite/Webpack configuration issues
- Module resolution failures
- After pulling changes that break the build

## Diagnostic Commands Run

```bash
# Primary build check
npm run build

# Vue SFC type check
vue-tsc --noEmit

# Vite specific (if using Vite)
npx vite build --debug 2>&1 | tail -100

# Webpack/vue-cli specific (if using webpack)
npx vue-cli-service build 2>&1

# Check dependencies
npm ls vue vue-router pinia vuex
```

## Example Session

```text
User: /vue-build

Agent:
# Vue Build Resolution

## Initial Diagnostics

```bash
$ npm run build
src/components/UserList.vue:15:10 - error TS2304: Cannot find name 'users'.
src/utils/api.ts:8:5 - error TS2322: Type 'string' is not assignable to type 'number'.
src/App.vue:25:3 - error Vue: Property 'items' was accessed during render but is not defined
```

Errors found: 3

## Fix 1: Undefined Variable in Template

File: src/components/UserList.vue:15
Error: Property 'users' was accessed during render but is not defined
Cause: Missing in defineProps

```vue
<!-- Changed -->
<template>
  <div v-for="user in users" :key="user.id">
    {{ user.name }}
  </div>
</template>

<script setup lang="ts">
// Added missing props
const props = defineProps<{
  users: User[]
}>()
</script>
```

```bash
$ npm run build
# 2 errors remaining
```

## Fix 2: Type Mismatch

File: src/utils/api.ts:8
Error: Type 'string' is not assignable to type 'number'

```typescript
// Changed
const userId: number = response.data.id  // id is string

// To
const userId: number = parseInt(response.data.id, 10)
```

```bash
$ npm run build
# 1 error remaining
```

## Fix 3: Missing Import

File: src/App.vue:25
Error: Cannot find name 'items'

```vue
<script setup lang="ts">
// Added missing import
import { items } from './data/items'
</script>
```

```bash
$ npm run build
# Build successful!
```

## Final Verification

```bash
$ vue-tsc --noEmit
# No errors

$ npm run test
PASS src/components/UserList.test.ts
```

## Summary

| Metric | Count |
|--------|-------|
| Build errors fixed | 3 |
| Files modified | 3 |
| Remaining issues | 0 |

Build Status: ✅ SUCCESS
```

## Common Errors Fixed

| Error | Typical Fix |
|-------|-------------|
| `Property 'X' was accessed during render but is not defined` | Add to defineProps/data |
| `Cannot find module './X.vue'` | Add Vue type declaration |
| `v-model value must be a valid JavaScript expression` | Fix v-model syntax |
| `Failed to resolve import` | Fix import path or add alias |
| `Unexpected token` | Add Vite plugin or loader |
| `Type 'X' is not assignable to type 'Y'` | Add type conversion |

## Fix Strategy

1. **SFC compilation errors first** - Templates must compile
2. **Type errors second** - TypeScript must pass
3. **Config errors third** - Build tools must work
4. **One fix at a time** - Verify each change
5. **Minimal changes** - Don't refactor, just fix

## Stop Conditions

The agent will stop and report if:
- Same error persists after 3 attempts
- Fix introduces more errors
- Requires architectural changes
- Missing external dependencies
- Vue 2 to Vue 3 migration required

## Related Commands

- `/vue-test` - Run tests after build succeeds
- `/vue-review` - Review code quality
- `/typescript-build` - Fix TypeScript-only errors
- `/verify` - Full verification loop

## Related

- Agent: `agents/vue-build-resolver.md`
- Skills: `skills/vue-patterns/`, `skills/vue-testing/`
