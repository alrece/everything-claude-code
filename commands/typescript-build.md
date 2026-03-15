---
description: Fix TypeScript build errors, type errors, module resolution issues, and tsconfig problems incrementally. Invokes the typescript-build-resolver agent for minimal, surgical fixes.
---

# TypeScript Build and Fix

This command invokes the **typescript-build-resolver** agent to incrementally fix TypeScript compilation errors with minimal changes.

## What This Command Does

1. **Run Diagnostics**: Execute `tsc --noEmit`, `npm run build`
2. **Parse Errors**: Group by file and sort by severity
3. **Fix Incrementally**: One error at a time
4. **Verify Each Fix**: Re-run type check after each change
5. **Report Summary**: Show what was fixed and what remains

## When to Use

Use `/typescript-build` when:
- `npx tsc --noEmit` fails with errors
- `npm run build` fails due to type errors
- Module resolution issues occur
- tsconfig.json configuration errors
- Type inference failures
- After pulling changes that break the build

## Diagnostic Commands Run

```bash
# Primary type check
npx tsc --noEmit --pretty

# Show all errors without cache
npx tsc --noEmit --pretty --incremental false

# Build check
npm run build

# ESLint TypeScript checks
npx eslint . --ext .ts,.tsx

# Check tsconfig
npx tsc --showConfig
```

## Example Session

```text
User: /typescript-build

Agent:
# TypeScript Build Resolution

## Initial Diagnostics

```bash
$ npx tsc --noEmit
src/utils/parser.ts:15:7 - error TS7006: Parameter 'data' implicitly has an 'any' type.
src/services/api.ts:28:5 - error TS2322: Type 'string' is not assignable to type 'number'.
src/types/user.ts:8:3 - error TS2724: Module '"./constants"' has no exported member 'ROLES'.
```

Errors found: 3

## Fix 1: Implicit Any Type

File: src/utils/parser.ts:15
Error: Parameter 'data' implicitly has an 'any' type

```typescript
// Before
export function parse(data) {
  return JSON.parse(data)
}

// After
export function parse(data: string): unknown {
  return JSON.parse(data)
}
```

```bash
$ npx tsc --noEmit
# 2 errors remaining
```

## Fix 2: Type Mismatch

File: src/services/api.ts:28
Error: Type 'string' is not assignable to type 'number'

```typescript
// Before
const timeout: number = process.env.TIMEOUT  // string!

// After
const timeout: number = parseInt(process.env.TIMEOUT || '5000', 10)
```

```bash
$ npx tsc --noEmit
# 1 error remaining
```

## Fix 3: Missing Export

File: src/types/user.ts:8
Error: Module '"./constants"' has no exported member 'ROLES'

```typescript
// Before (in constants.ts)
export const UserRole = { Admin: 'admin', User: 'user' }

// After - Added export
export const UserRole = { Admin: 'admin', User: 'user' }
export const ROLES = Object.values(UserRole)
```

```bash
$ npx tsc --noEmit
# No errors!
```

## Final Verification

```bash
$ npm run build
Build successful!

$ npm run test
PASS src/utils/parser.test.ts
PASS src/services/api.test.ts
```

## Summary

| Metric | Count |
|--------|-------|
| Type errors fixed | 3 |
| Files modified | 3 |
| Remaining issues | 0 |

Build Status: ✅ SUCCESS
```

## Common Errors Fixed

| Error | Typical Fix |
|-------|-------------|
| `implicitly has 'any' type` | Add explicit type annotation |
| `Type 'X' is not assignable to type 'Y'` | Type conversion or fix source type |
| `Object is possibly 'undefined'` | Add null check or optional chaining |
| `Cannot find module 'X'` | Install package or fix import path |
| `Cannot find name 'X'` | Add import or declare variable |
| `Property 'X' does not exist on type` | Extend interface or fix property name |
| `Cannot find declaration file` | Install `@types/package` |
| `Generic type requires type argument` | Provide type argument |

## Fix Strategy

1. **Implicit any errors first** - Add type annotations
2. **Type mismatches second** - Fix type assignments
3. **Module errors third** - Fix imports/exports
4. **One fix at a time** - Verify each change
5. **Minimal changes** - Don't refactor, just fix

## Stop Conditions

The agent will stop and report if:
- Same error persists after 3 attempts
- Fix introduces more errors
- Requires architectural changes
- Missing external dependencies
- Type system limitation requires redesign

## Quick Reference

### Adding Type Annotations

```typescript
// Parameter types
function greet(name: string): string { return `Hello ${name}` }

// Variable types
const count: number = 42
const items: string[] = ['a', 'b', 'c']
const map: Map<string, number> = new Map()

// Generic types
const result: Result<string, Error> = { ok: true, value: 'success' }
```

### Fixing Type Mismatches

```typescript
// Type conversion
const num: number = parseInt(str, 10)
const str: string = num.toString()

// Type assertion (use sparingly)
const el = document.getElementById('app') as HTMLDivElement

// Type guards
if (typeof value === 'string') { /* value is string */ }
if ('length' in value) { /* value has length property */ }
```

### Module Resolution

```typescript
// Missing declaration - create .d.ts
declare module 'untyped-package' {
  export function doSomething(input: string): number
}

// Path alias in tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  }
}
```

## Related Commands

- `/typescript-test` - Run tests after build succeeds
- `/typescript-review` - Review code quality
- `/vue-build` - Fix Vue + TypeScript errors
- `/verify` - Full verification loop

## Related

- Agent: `agents/typescript-build-resolver.md`
- Skills: `skills/typescript-patterns/`
