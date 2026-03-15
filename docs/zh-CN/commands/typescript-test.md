---
description: 为TypeScript强制执行TDD工作流程。使用Vitest/Jest首先编写测试，然后实现。验证80%以上的覆盖率并保持类型安全。
---

# TypeScript TDD 命令

此命令使用 Vitest 或 Jest 并保持完整类型安全，为 TypeScript 代码强制执行测试驱动开发方法。

## 此命令的作用

1. **定义类型/接口**：搭建带有正确类型的函数签名
2. **编写类型化测试**：创建全面的测试用例（RED 阶段）
3. **运行测试**：验证测试因正确原因而失败
4. **实现代码**：编写最少的代码以通过测试（GREEN 阶段）
5. **重构**：改进代码，同时保持测试通过
6. **检查覆盖率**：确保 80% 以上的覆盖率

## 何时使用

在以下情况下使用 `/typescript-test`：

* 实现新的 TypeScript 函数时
* 为现有代码添加测试覆盖率时
* 修复 bug 时（先编写失败的测试）
* 构建关键业务逻辑时
* 学习 TypeScript 中的 TDD 工作流程时

## TDD 循环

```
RED     → 编写带有正确类型的失败测试
GREEN   → 实现最少的代码以通过测试
REFACTOR → 改进代码，测试保持通过
REPEAT  → 下一个测试用例
```

## 示例会话

````text
User: /typescript-test I need a function to validate email addresses

Agent:
# TDD Session: Email Validator

## 步骤 1：定义类型

```typescript
// src/validators/email.ts

export interface ValidationResult {
  valid: boolean
  error?: string
}

export type EmailValidator = (email: string) => ValidationResult
```

## 步骤 2：编写测试（RED 阶段）

```typescript
// src/validators/email.test.ts
import { describe, it, expect } from 'vitest'
import type { ValidationResult } from './email'

// 这将失败，因为函数还不存在
import { validateEmail } from './email'

describe('validateEmail', () => {
  // 有效的邮箱
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

  // 无效的邮箱
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

  // 边界情况
  it('handles null-like input gracefully', () => {
    // 类型安全测试
    const result = validateEmail('' as string)
    expect(result).toMatchObject<ValidationResult>({
      valid: expect.any(Boolean),
      error: expect.any(String)
    })
  })
})
```

## 步骤 3：运行测试 - 验证失败

```bash
$ npm run test

FAIL src/validators/email.test.ts
  ✗ validateEmail > accepts user@example.com (simple email)
    Error: Cannot find module './email'
```

✓ 测试按预期失败（模块未找到）。

## 步骤 4：实现最少的代码（GREEN 阶段）

```typescript
// src/validators/email.ts

export interface ValidationResult {
  valid: boolean
  error?: string
}

// Email regex pattern
const EMAIL_REGEX = /^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/

export function validateEmail(email: string): ValidationResult {
  // 空值检查
  if (!email || email.length === 0) {
    return { valid: false, error: 'Email cannot be empty' }
  }

  // 不能有空格
  if (email.includes(' ')) {
    return { valid: false, error: 'Email cannot contain spaces' }
  }

  // 必须恰好有一个 @
  const atCount = email.split('@').length - 1
  if (atCount !== 1) {
    return {
      valid: false,
      error: atCount === 0 ? 'Email must contain @' : 'Email must contain exactly one @'
    }
  }

  const [localPart, domain] = email.split('@')

  // 本地部分验证
  if (!localPart || localPart.length === 0) {
    return { valid: false, error: 'Email must have a local part before @' }
  }

  // 域名验证
  if (!domain || domain.length === 0) {
    return { valid: false, error: 'Email must have a domain after @' }
  }

  // 必须有 TLD
  if (!domain.includes('.')) {
    return { valid: false, error: 'Email domain must have a TLD' }
  }

  // 完整模式匹配
  if (!EMAIL_REGEX.test(email)) {
    return { valid: false, error: 'Email format is invalid' }
  }

  return { valid: true }
}
```

## 步骤 5：运行测试 - 验证通过

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

✓ 所有测试通过！

## 步骤 6：检查覆盖率

```bash
$ npm run test -- --coverage

PASS src/validators/email.test.ts
Coverage: 100% Statements | 100% Branches | 100% Functions | 100% Lines
```

✓ 覆盖率：100%

## TDD 完成！
````

## 测试模式

### 类型化测试辅助函数

```typescript
// 类型安全的测试夹具
function createMockUser(overrides?: Partial<User>): User {
  return {
    id: 1,
    name: 'Test User',
    email: 'test@example.com',
    ...overrides
  }
}

// 类型安全的 mock 函数
const mockFetch = vi.fn<() => Promise<Response>>()
```

### 参数化测试

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

### 类型测试

```typescript
import { expectTypeOf } from 'expect-type'

it('returns correct types', () => {
  expectTypeOf(validateEmail('test@example.com')).toMatchTypeOf<ValidationResult>()
  expectTypeOf(validateEmail).parameters.toEqualTypeOf<[string]>()
  expectTypeOf(validateEmail).returns.toEqualTypeOf<ValidationResult>()
})
```

### 测试异步代码

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

### 带类型的 Mock

```typescript
import { vi } from 'vitest'

// 类型安全的模块 mock
vi.mock('./api', () => ({
  fetchUser: vi.fn<(id: number) => Promise<User>>()
    .mockResolvedValue({ id: 1, name: 'Mock User' })
}))

// 类型安全的 spy
const consoleSpy = vi.spyOn(console, 'log')
  .mockImplementation(() => {})
```

## 覆盖率命令

```bash
# Vitest coverage
npm run test -- --coverage

# Jest coverage
npm run test -- --coverage --watchAll=false

# 详细覆盖率报告
npm run test -- --coverage --reporter=verbose

# 特定文件的覆盖率
npm run test -- --coverage src/utils/parser.ts
```

## 覆盖率目标

| 代码类型 | 目标 |
|-----------|--------|
| 关键业务逻辑 | 100% |
| 公共 API | 90%+ |
| 通用代码 | 80%+ |
| 生成的代码 | 排除 |
| 类型定义 | N/A |

## TDD 最佳实践

**应该做：**

* 先编写测试，再编写任何实现
* 编写测试前先定义类型
* 测试行为，而非实现细节
* 包含边界情况（null、undefined、空值、最大值）
* 使用 `expectTypeOf` 进行类型级别测试
* 保持测试类型安全（避免 `as any`）

**不应该做：**

* 在编写测试之前编写实现
* 跳过 RED 阶段
* 在测试中使用 `any`
* 直接测试私有方法
* 忽略测试中的 TypeScript 错误

## 相关命令

* `/typescript-build` - 修复构建错误
* `/typescript-review` - 在实现后审查代码
* `/verify` - 运行完整的验证循环

## 相关

* 技能：`skills/typescript-testing/`
* 技能：`skills/tdd-workflow/`
