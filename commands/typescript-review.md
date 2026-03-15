---
description: Comprehensive TypeScript code review for type safety, generics, strict mode compliance, and advanced patterns. Invokes the typescript-reviewer agent.
---

# TypeScript Code Review

This command invokes the **typescript-reviewer** agent for comprehensive TypeScript-specific code review.

## What This Command Does

1. **Identify TypeScript Changes**: Find modified `.ts`, `.tsx` files via `git diff`
2. **Run Type Check**: Execute `tsc --noEmit` for type errors
3. **Type Safety Review**: Check for `any` usage, type assertions, unsafe patterns
4. **Strict Mode Check**: Verify strict mode compliance
5. **Generic Review**: Analyze type parameters and constraints
6. **Generate Report**: Categorize issues by severity

## When to Use

Use `/typescript-review` when:
- After writing or modifying TypeScript code
- Before committing TypeScript changes
- Reviewing pull requests with TypeScript code
- Onboarding to a new TypeScript codebase
- Learning TypeScript best practices

## Review Categories

### CRITICAL (Must Fix)
- `any` type without justification
- Type assertions without runtime validation
- Non-null assertions (`!`) without null checks
- Unsafe type guards
- Hardcoded credentials or secrets

### HIGH (Should Fix)
- Implicit any in parameters
- Strict null check violations
- Missing return types on exported functions
- Under-constrained generics
- Unused type parameters

### MEDIUM (Consider)
- Missing JSDoc on public APIs
- Interface vs type alias misuse
- Excessive type assertions
- Missing readonly modifiers
- Overly complex conditional types

## Automated Checks Run

```bash
# Type check
npx tsc --noEmit --pretty

# Show all errors
npx tsc --noEmit --pretty --incremental false

# ESLint TypeScript rules
npx eslint . --ext .ts,.tsx

# Check for unused exports
npx ts-prune

# Find circular dependencies
npx madge --circular src/
```

## Example Usage

```text
User: /typescript-review

Agent:
# TypeScript Code Review Report

## Files Reviewed
- src/services/api.ts (modified)
- src/utils/parser.ts (modified)
- src/types/models.ts (modified)

## Type Check Results
✓ tsc --noEmit: No errors
✓ ESLint: No issues

## Issues Found

[CRITICAL] Unvalidated Type Assertion
File: src/utils/parser.ts:28
Issue: Type assertion without runtime validation
```typescript
// BAD: Unvalidated assertion
const user = JSON.parse(data) as User
```
Fix: Validate before assertion
```typescript
// GOOD: Validate then assert
import { z } from 'zod'
const UserSchema = z.object({
  id: z.number(),
  name: z.string(),
  email: z.string().email()
})

const user = UserSchema.parse(JSON.parse(data)) as User
```

[HIGH] Under-Constrained Generic
File: src/services/api.ts:45
Issue: Generic type parameter `T` is not constrained
```typescript
// BAD: No constraint
function getProperty<T, K>(obj: T, key: K) {
  return obj[key]
}
```
Fix: Add proper constraints
```typescript
// GOOD: Properly constrained
function getProperty<T extends object, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key]
}
```

[HIGH] Non-Null Assertion Without Check
File: src/services/api.ts:62
Issue: Using `!` without null check
```typescript
// BAD: Assumption without check
const name = user!.profile!.name
```
Fix: Use optional chaining
```typescript
// GOOD: Safe access
const name = user?.profile?.name ?? 'Unknown'
```

## Summary
- CRITICAL: 1
- HIGH: 2
- MEDIUM: 0

Recommendation: ❌ Block merge until CRITICAL issue is fixed
```

## Approval Criteria

| Status | Condition |
|--------|-----------|
| ✅ Approve | No CRITICAL or HIGH issues |
| ⚠️ Warning | Only MEDIUM issues (merge with caution) |
| ❌ Block | CRITICAL or HIGH issues found |

## Strict Mode Checks

When `strict: true` in tsconfig.json, verify:

| Option | Check |
|--------|-------|
| `noImplicitAny` | No missing type annotations |
| `strictNullChecks` | All nullable values handled |
| `strictFunctionTypes` | Function parameter types correct |
| `noImplicitReturns` | All code paths return value |
| `noUncheckedIndexedAccess` | Index access checked for undefined |

## Generic Best Practices

### Do
```typescript
// Constrain generics appropriately
function getProperty<T extends object, K extends keyof T>(obj: T, key: K): T[K]

// Use defaults for optional generics
type Result<T = unknown, E = Error> = { ok: boolean; value?: T; error?: E }

// Use infer for conditional types
type UnwrapPromise<T> = T extends Promise<infer U> ? U : T
```

### Don't
```typescript
// Overly permissive
function process<T = any>(data: T): T

// Unused type parameter
function map<T>(items: unknown[]): T[]

// Impossible constraint
function sort<T extends string | number | boolean>(items: T[])
```

## Integration with Other Commands

- Use `/typescript-test` first to ensure tests pass
- Use `/typescript-build` if build errors occur
- Use `/typescript-review` before committing
- Use `/code-review` for non-TypeScript specific concerns

## Related

- Agent: `agents/typescript-reviewer.md`
- Skills: `skills/typescript-patterns/`, `skills/typescript-testing/`
