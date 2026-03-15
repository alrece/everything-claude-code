---
description: 逐步修复Vue构建错误、SFC编译问题和Vite/Webpack配置问题。调用vue-build-resolver代理进行最小化、精确的修复。
---

# Vue 构建与修复

此命令调用 **vue-build-resolver** 代理，以最小的更改增量修复 Vue 构建错误。

## 此命令的作用

1. **运行诊断**：执行 `npm run build`、`vue-tsc --noEmit`
2. **解析错误**：按文件分组并按严重性排序
3. **增量修复**：一次修复一个错误
4. **验证每次修复**：每次更改后重新运行构建
5. **报告摘要**：显示已修复的内容和剩余问题

## 何时使用

在以下情况使用 `/vue-build`：

* `npm run build` 因错误而失败
* Vue SFC 编译错误
* `vue-tsc` 报告类型错误
* Vite/Webpack 配置问题
* 模块解析失败
* 拉取更改后导致构建失败

## 运行的诊断命令

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

## 示例会话

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

发现错误：3

## 修复 1：模板中未定义的变量

文件：src/components/UserList.vue:15
错误：Property 'users' was accessed during render but is not defined
原因：defineProps 中缺少

```vue
<!-- 修改后 -->
<template>
  <div v-for="user in users" :key="user.id">
    {{ user.name }}
  </div>
</template>

<script setup lang="ts">
// 添加缺少的 props
const props = defineProps<{
  users: User[]
}>()
</script>
```

```bash
$ npm run build
# 剩余 2 个错误
```

## 修复 2：类型不匹配

文件：src/utils/api.ts:8
错误：Type 'string' is not assignable to type 'number'

```typescript
// 修改前
const userId: number = response.data.id  // id 是 string

// 修改后
const userId: number = parseInt(response.data.id, 10)
```

```bash
$ npm run build
# 剩余 1 个错误
```

## 修复 3：缺少导入

文件：src/App.vue:25
错误：Cannot find name 'items'

```vue
<script setup lang="ts">
// 添加缺少的导入
import { items } from './data/items'
</script>
```

```bash
$ npm run build
# 构建成功！
```

## 最终验证

```bash
$ vue-tsc --noEmit
# 无错误

$ npm run test
PASS src/components/UserList.test.ts
```

## 摘要

| 指标 | 数量 |
|--------|-------|
| 已修复的构建错误 | 3 |
| 已修改的文件 | 3 |
| 剩余问题 | 0 |

构建状态：✅ 成功
```

## 常见错误修复

| 错误 | 典型修复 |
|-------|-------------|
| `Property 'X' was accessed during render but is not defined` | 添加到 defineProps/data |
| `Cannot find module './X.vue'` | 添加 Vue 类型声明 |
| `v-model value must be a valid JavaScript expression` | 修复 v-model 语法 |
| `Failed to resolve import` | 修复导入路径或添加别名 |
| `Unexpected token` | 添加 Vite 插件或 loader |
| `Type 'X' is not assignable to type 'Y'` | 添加类型转换 |

## 修复策略

1. **SFC 编译错误优先** - 模板必须能编译
2. **类型错误其次** - TypeScript 必须通过
3. **配置错误第三** - 构建工具必须工作
4. **一次修复一个** - 验证每次更改
5. **最小化更改** - 不要重构，只修复

## 停止条件

代理将在以下情况下停止并报告：
- 同一错误在 3 次尝试后仍然存在
- 修复引入更多错误
- 需要架构更改
- 缺少外部依赖
- 需要 Vue 2 到 Vue 3 迁移

## 相关命令

- `/vue-test` - 构建成功后运行测试
- `/vue-review` - 审查代码质量
- `/typescript-build` - 仅修复 TypeScript 错误
- `/verify` - 完整验证循环

## 相关

- Agent: `agents/vue-build-resolver.md`
- Skills: `skills/vue-patterns/`, `skills/vue-testing/`
