---
description: Enforce TDD workflow for TypeScript. Write tests first with Vitest/Jest, then implement. Verify 80%+ coverage with type-safe tests.
---

# TypeScript TDD Command

This command enforces test-driven development methodology for TypeScript code using Vitest or Jest with full type safety.

## What This Command Does

1. **Define Types/Interfaces**: Scaffold function signatures with proper types
2. **Write Typed Tests**: Create comprehensive test cases (RED)
3. **Run Tests**: Verify tests fail for the right reason
4. **Implement Code**: Write minimal code to pass (GREEN)
5. **Refactor**: Improve while keeping tests green
6. **Check Coverage**: Ensure 80%+ coverage

## When to Use

Use `/typescript-test` when:
- Implementing new TypeScript functions
- Adding test coverage to existing code
- Fixing bugs (write failing test first)
- Building critical business logic
- Learning TDD workflow in TypeScript

## TDD Cycle

```
RED     → Write failing test with proper types
GREEN   → Implement minimal code to pass
REFACTOR → Improve code, tests stay green
REPEAT  → Next test case
```

## Example Session

````text
User: /typescript-test I need a function to validate email addresses

Agent:
# TDD Session: Email Validator

## Step 1: Define Types

```typescript
// src/validators/email.ts

export interface ValidationResult {
  valid: boolean
  error?: string
}

export type EmailValidator = (email: string) => ValidationResult
```

## Step 2: Write Tests (RED)

```typescript
// src/validators/email.test.ts
import { describe, it, expect } from 'vitest'
import type { ValidationResult } from './email'

// This will fail because function doesn't exist yet
import { validateEmail } from './email'

describe('validateEmail', () => {
  // Valid emails
  it.each([
    ['user@example.com', 'simple email'],
    ['user@mail.example.com', 'with subdomain'],
    ['user+tag@example.com', 'with plus sign'],
    ['first.last@example.com', 'with dots'],
    ['USER@EXAMPLE.COM', 'uppercase domain'],
  ])('accepts %s (%s)', (email) => {
    const result = validateEmail(email)
    expect(result.valid).toBe(true)
    expect(result.error).toBeUndefined()
  })

  // Invalid emails
  it.each([
    ['', 'empty string'],
    ['userexample.com', 'no at sign'],
    ['user@', 'no domain'],
    ['@example.com', 'no local part'],
    ['user@@example.com', 'double at'],
    ['user @example.com', 'contains spaces'],
    ['user@example', 'no TLD'],
  ])('rejects %s (%s)', (email) => {
    const result = validateEmail(email)
    expect(result.valid).toBe(false)
    expect(result.error).toBeDefined()
  })

  // Edge cases
  it('handles null-like input gracefully', () => {
    // Type safety test
    const result = validateEmail('' as string)
    expect(result).toMatchObject<ValidationResult>({
      valid: expect.any(Boolean),
      error: expect.any(String)
    })
  })
})
```

## Step 3: Run Tests - Verify FAIL

```bash
$ npm run test

FAIL src/validators/email.test.ts
  ✗ validateEmail > accepts user@example.com (simple email)
    Error: Cannot find module './email'
```

✓ Tests fail as expected (module not found).

## Step 4: Implement Minimal Code (GREEN)

```typescript
// src/validators/email.ts

export interface ValidationResult {
  valid: boolean
  error?: string
}

// Email regex pattern
const EMAIL_REGEX = /^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/

export function validateEmail(email: string): ValidationResult {
  // Empty check
  if (!email || email.length === 0) {
    return { valid: false, error: 'Email cannot be empty' }
  }

  // No spaces
  if (email.includes(' ')) {
    return { valid: false, error: 'Email cannot contain spaces' }
  }

  // Must have exactly one @
  const atCount = email.split('@').length - 1
  if (atCount !== 1) {
    return {
      valid: false,
      error: atCount === 0 ? 'Email must contain @' : 'Email must contain exactly one @'
    }
  }

  const [localPart, domain] = email.split('@')

  // Local part validation
  if (!localPart || localPart.length === 0) {
    return { valid: false, error: 'Email must have a local part before @' }
  }

  // Domain validation
  if (!domain || domain.length === 0) {
    return { valid: false, error: 'Email must have a domain after @' }
  }

  // Must have TLD
  if (!domain.includes('.')) {
    return { valid: false, error: 'Email domain must have a TLD' }
  }

  // Full pattern match
  if (!EMAIL_REGEX.test(email)) {
    return { valid: false, error: 'Email format is invalid' }
  }

  return { valid: true }
}
```

## Step 5: Run Tests - Verify PASS

```bash
$ npm run test

PASS src/validators/email.test.ts
  ✓ validateEmail > accepts user@example.com (simple email)
  ✓ validateEmail > accepts user@mail.example.com (with subdomain)
  ✓ validateEmail > accepts user+tag@example.com (with plus sign)
  ✓ validateEmail > accepts first.last@example.com (with dots)
  ✓ validateEmail > accepts USER@EXAMPLE.COM (uppercase domain)
  ✓ validateEmail > rejects empty string (empty string)
  ✓ validateEmail > rejects userexample.com (no at sign)
  ✓ validateEmail > rejects user@ (no domain)
  ✓ validateEmail > rejects @example.com (no local part)
  ✓ validateEmail > rejects user@@example.com (double at)
  ✓ validateEmail > rejects user @example.com (contains spaces)
  ✓ validateEmail > rejects user@example (no TLD)
  ✓ validateEmail > handles null-like input gracefully

PASS 13/13 tests
```

✓ All tests passing!

## Step 6: Check Coverage

```bash
$ npm run test -- --coverage

PASS src/validators/email.test.ts
Coverage: 100% Statements | 100% Branches | 100% Functions | 100% Lines
```

✓ Coverage: 100%

## TDD Complete!
````

## Test Patterns

### Typed Test Helpers

```typescript
// Type-safe test fixtures
function createMockUser(overrides?: Partial<User>): User {
  return {
    id: 1,
    name: 'Test User',
    email: 'test@example.com',
    ...overrides
  }
}

// Type-safe mock functions
const mockFetch = vi.fn<() => Promise<Response>>()
```

### Parameterized Tests

```typescript
// Vitest
it.each([
  [1, 2, 3],
  [2, 3, 5],
  [3, 5, 8],
])('add(%i, %i) = %i', (a, b, expected) => {
  expect(add(a, b)).toBe(expected)
})

// Jest
test.each([
  [1, 2, 3],
  [2, 3, 5],
])('add(%i, %i) = %i', (a, b, expected) => {
  expect(add(a, b)).toBe(expected)
})
```

### Type Testing

```typescript
import { expectTypeOf } from 'expect-type'

it('returns correct types', () => {
  expectTypeOf(validateEmail('test@example.com')).toMatchTypeOf<ValidationResult>()
  expectTypeOf(validateEmail).parameters.toEqualTypeOf<[string]>()
  expectTypeOf(validateEmail).returns.toEqualTypeOf<ValidationResult>()
})
```

### Testing Async Code

```typescript
describe('async operations', () => {
  it('handles async success', async () => {
    const result = await fetchData<number>('url')
    expectTypeOf(result).toBeNumber()
    expect(result).toBe(42)
  })

  it('handles async errors', async () => {
    await expect(fetchData('invalid')).rejects.toThrow('Network error')
  })
})
```

### Mocking with Types

```typescript
import { vi } from 'vitest'

// Type-safe module mock
vi.mock('./api', () => ({
  fetchUser: vi.fn<(id: number) => Promise<User>>()
    .mockResolvedValue({ id: 1, name: 'Mock User' })
}))

// Type-safe spy
const consoleSpy = vi.spyOn(console, 'log')
  .mockImplementation(() => {})
```

## Coverage Commands

```bash
# Vitest coverage
npm run test -- --coverage

# Jest coverage
npm run test -- --coverage --watchAll=false

# Detailed coverage report
npm run test -- --coverage --reporter=verbose

# Coverage for specific file
npm run test -- --coverage src/utils/parser.ts
```

## Coverage Targets

| Code Type | Target |
|-----------|--------|
| Critical business logic | 100% |
| Public APIs | 90%+ |
| General code | 80%+ |
| Generated code | Exclude |
| Type definitions | N/A |

## TDD Best Practices

**DO:**
- Write test FIRST, before any implementation
- Define types before writing tests
- Test behavior, not implementation details
- Include edge cases (null, undefined, empty, max values)
- Use `expectTypeOf` for type-level testing
- Keep tests type-safe (avoid `as any`)

**DON'T:**
- Write implementation before tests
- Skip the RED phase
- Use `any` in tests
- Test private methods directly
- Ignore TypeScript errors in tests

## Related Commands

- `/typescript-build` - Fix build errors
- `/typescript-review` - Review code after implementation
- `/verify` - Run full verification loop

## Related

- Skill: `skills/typescript-testing/`
- Skill: `skills/tdd-workflow/`
