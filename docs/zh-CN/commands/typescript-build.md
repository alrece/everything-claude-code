---
description: 逐步修复TypeScript构建错误、类型错误、模块解析问题和tsconfig配置问题。调用typescript-build-resolver代理进行最小化、精确的修复。
---

# TypeScript 构建与修复

此命令调用 **typescript-build-resolver** 代理，以最小的更改增量修复 TypeScript 编译错误。

## 此命令的作用

1. **运行诊断**：执行 `tsc --noEmit`、`npm run build`
2. **解析错误**：按文件分组并按严重性排序
3. **增量修复**：一次修复一个错误
4. **验证每次修复**：每次更改后重新运行类型检查
5. **报告摘要**：显示已修复的内容和剩余问题

## 何时使用

在以下情况使用 `/typescript-build`：

* `npx tsc --noEmit` 因错误而失败
* `npm run build` 因类型错误而失败
* 模块解析问题
* tsconfig.json 配置错误
* 类型推断失败
* 拉取更改后导致构建失败

## 运行的诊断命令

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

## 示例会话

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

发现错误：3

## 修复 1：隐式 Any 类型

文件：src/utils/parser.ts:15
错误：Parameter 'data' implicitly has an 'any' type

```typescript
// 修改前
export function parse(data) {
  return JSON.parse(data)
}

// 修改后
export function parse(data: string): unknown {
  return JSON.parse(data)
}
```

```bash
$ npx tsc --noEmit
# 剩余 2 个错误
```

## 修复 2：类型不匹配

文件：src/services/api.ts:28
错误：Type 'string' is not assignable to type 'number'

```typescript
// 修改前
const timeout: number = process.env.TIMEOUT  // string!

// 修改后
const timeout: number = parseInt(process.env.TIMEOUT || '5000', 10)
```

```bash
$ npx tsc --noEmit
# 剩余 1 个错误
```

## 修复 3：缺少导出

文件：src/types/user.ts:8
错误：Module '"./constants"' has no exported member 'ROLES'

```typescript
// 修改前 (in constants.ts)
export const UserRole = { Admin: 'admin', User: 'user' }

// 修改后 - 添加导出
export const UserRole = { Admin: 'admin', User: 'user' }
export const ROLES = Object.values(UserRole)
```

```bash
$ npx tsc --noEmit
# 无错误！
```

## 最终验证

```bash
$ npm run build
Build successful!

$ npm run test
PASS src/utils/parser.test.ts
PASS src/services/api.test.ts
```

## 摘要

| 指标 | 数量 |
|--------|-------|
| 已修复的类型错误 | 3 |
| 已修改的文件 | 3 |
| 剩余问题 | 0 |

构建状态：✅ 成功
```

## 常见错误修复

| 错误 | 典型修复 |
|-------|-------------|
| `implicitly has 'any' type` | 添加显式类型注解 |
| `Type 'X' is not assignable to type 'Y'` | 类型转换或修复源类型 |
| `Object is possibly 'undefined'` | 添加 null 检查或可选链 |
| `Cannot find module 'X'` | 安装包或修复导入路径 |
| `Cannot find name 'X'` | 添加导入或声明变量 |
| `Property 'X' does not exist on type` | 扩展接口或修复属性名 |
| `Cannot find declaration file` | 安装 `@types/package` |
| `Generic type requires type argument` | 提供类型参数 |

## 修复策略

1. **隐式 any 错误优先** - 添加类型注解
2. **类型不匹配其次** - 修复类型赋值
3. **模块错误第三** - 修复导入/导出
4. **一次修复一个** - 验证每次更改
5. **最小化更改** - 不要重构，只修复

## 停止条件

代理将在以下情况下停止并报告：
- 同一错误在 3 次尝试后仍然存在
- 修复引入更多错误
- 需要架构更改
- 缺少外部依赖
- 类型系统限制需要重新设计

## 快速参考

### 添加类型注解

```typescript
// 参数类型
function greet(name: string): string { return `Hello ${name}` }

// 变量类型
const count: number = 42
const items: string[] = ['a', 'b', 'c']
const map: Map<string, number> = new Map()

// 泛型类型
const result: Result<string, Error> = { ok: true, value: 'success' }
```

### 修复类型不匹配

```typescript
// 类型转换
const num: number = parseInt(str, 10)
const str: string = num.toString()

// 类型断言（谨慎使用）
const el = document.getElementById('app') as HTMLDivElement

// 类型守卫
if (typeof value === 'string') { /* value is string */ }
if ('length' in value) { /* value has length property */ }
```

### 模块解析

```typescript
// 缺少声明 - 创建 .d.ts
declare module 'untyped-package' {
  export function doSomething(input: string): number
}

// tsconfig.json 中的路径别名
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  }
}
```

## 相关命令

- `/typescript-test` - 构建成功后运行测试
- `/typescript-review` - 审查代码质量
- `/vue-build` - 修复 Vue + TypeScript 错误
- `/verify` - 完整验证循环

## 相关

- Agent: `agents/typescript-build-resolver.md`
- Skills: `skills/typescript-patterns/`
