---
description: 全面的TypeScript代码审查，涵盖类型安全、泛型、严格模式合规性和高级模式。调用typescript-reviewer代理。
---

# TypeScript 代码审查

此命令调用 **typescript-reviewer** 代理进行全面的 TypeScript 特定代码审查。

## 此命令的作用

1. **识别 TypeScript 变更**：通过 `git diff` 查找修改过的 `.ts`、`.tsx` 文件
2. **运行类型检查**：执行 `tsc --noEmit` 检查类型错误
3. **类型安全审查**：检查 `any` 使用、类型断言、不安全模式
4. **严格模式检查**：验证严格模式合规性
5. **泛型审查**：分析类型参数和约束
6. **生成报告**：按严重程度分类问题

## 何时使用

在以下情况使用 `/typescript-review`：

* 编写或修改 TypeScript 代码之后
* 提交 TypeScript 变更之前
* 审查包含 TypeScript 代码的拉取请求时
* 接手新的 TypeScript 代码库时
* 学习 TypeScript 最佳实践时

## 审查类别

### 严重（必须修复）

* 没有正当理由使用 `any` 类型
* 没有运行时验证的类型断言
* 没有 null 检查的非空断言（`!`）
* 不安全的类型守卫
* 硬编码的凭证或密钥

### 高（应该修复）

* 参数中隐式 any
* 严格 null 检查违规
* 导出函数缺少返回类型
* 泛型约束不足
* 未使用的类型参数

### 中（考虑修复）

* 公共 API 缺少 JSDoc
* Interface vs type alias 误用
* 过多的类型断言
* 缺少 readonly 修饰符
* 过于复杂的条件类型

## 运行的自动化检查

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

## 使用示例

```text
User: /typescript-review

Agent:
# TypeScript 代码审查报告

## 已审查的文件
- src/services/api.ts (modified)
- src/utils/parser.ts (modified)
- src/types/models.ts (modified)

## 类型检查结果
✓ tsc --noEmit: 无错误
✓ ESLint: 无问题

## 发现的问题

[严重] 未验证的类型断言
文件：src/utils/parser.ts:28
问题：类型断言没有运行时验证
```typescript
// 错误：未验证的断言
const user = JSON.parse(data) as User
```
修复：断言前先验证
```typescript
// 正确：验证后断言
import { z } from 'zod'
const UserSchema = z.object({
  id: z.number(),
  name: z.string(),
  email: z.string().email()
})

const user = UserSchema.parse(JSON.parse(data)) as User
```

[高] 泛型约束不足
文件：src/services/api.ts:45
问题：泛型类型参数 `T` 没有约束
```typescript
// 错误：没有约束
function getProperty<T, K>(obj: T, key: K) {
  return obj[key]
}
```
修复：添加适当的约束
```typescript
// 正确：适当约束
function getProperty<T extends object, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key]
}
```

[高] 非 null 断言没有检查
文件：src/services/api.ts:62
问题：使用 `!` 没有 null 检查
```typescript
// 错误：没有检查的假设
const name = user!.profile!.name
```
修复：使用可选链
```typescript
// 正确：安全访问
const name = user?.profile?.name ?? 'Unknown'
```

## 摘要
- 严重：1
- 高：2
- 中：0

建议：❌ 在严重问题修复前阻止合并
```

## 审批标准

| 状态 | 条件 |
|--------|-----------|
| ✅ 通过 | 无严重或高问题 |
| ⚠️ 警告 | 仅中等问题（可谨慎合并） |
| ❌ 阻塞 | 存在严重或高问题 |

## 严格模式检查

当 tsconfig.json 中有 `strict: true` 时，验证：

| 选项 | 检查 |
|--------|-------|
| `noImplicitAny` | 没有缺少的类型注解 |
| `strictNullChecks` | 所有可空值已处理 |
| `strictFunctionTypes` | 函数参数类型正确 |
| `noImplicitReturns` | 所有代码路径都有返回值 |
| `noUncheckedIndexedAccess` | 索引访问已检查 undefined |

## 泛型最佳实践

### 应该做
```typescript
// 适当约束泛型
function getProperty<T extends object, K extends keyof T>(obj: T, key: K): T[K]

// 为可选泛型使用默认值
type Result<T = unknown, E = Error> = { ok: boolean; value?: T; error?: E }

// 在条件类型中使用 infer
type UnwrapPromise<T> = T extends Promise<infer U> ? U : T
```

### 不应该做
```typescript
// 过于宽松
function process<T = any>(data: T): T

// 未使用的类型参数
function map<T>(items: unknown[]): T[]

// 不可能的约束
function sort<T extends string | number | boolean>(items: T[])
```

## 与其他命令的集成

- 先使用 `/typescript-test` 确保测试通过
- 如果出现构建错误使用 `/typescript-build`
- 提交前使用 `/typescript-review`
- 对于非 TypeScript 特定问题使用 `/code-review`

## 相关

- Agent: `agents/typescript-reviewer.md`
- Skills: `skills/typescript-patterns/`, `skills/typescript-testing/`
