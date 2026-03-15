---
description: 全面的Vue代码审查，涵盖响应式模式、组件设计、Composition API最佳实践和性能优化。调用vue-reviewer代理。
---

# Vue 代码审查

此命令调用 **vue-reviewer** 代理进行全面的 Vue 特定代码审查。

## 此命令的作用

1. **识别 Vue 变更**：通过 `git diff` 查找修改过的 `.vue`、`.ts`、`.js` 文件
2. **运行静态分析**：执行 `vue-tsc`、ESLint Vue 规则
3. **响应式审查**：检查响应式陷阱、ref/reactive 误用
4. **组件审查**：分析 props、emits、slots 和组件结构
5. **性能检查**：识别不必要的重新渲染、内存泄漏
6. **生成报告**：按严重程度分类问题

## 何时使用

在以下情况使用 `/vue-review`：

* 编写或修改 Vue 组件之后
* 提交 Vue 变更之前
* 审查包含 Vue 代码的拉取请求时
* 接手新的 Vue 代码库时
* 学习 Vue 最佳实践时

## 审查类别

### 严重（必须修复）

* 子组件直接修改 props
* 未清理的定时器/监听器导致内存泄漏
* v-html 导致的 XSS 漏洞
* 循环组件依赖
* 模板中硬编码的凭证

### 高（应该修复）

* 解构导致响应性丢失
* props/emit 缺少类型定义
* watchEffect 没有清理函数
* 超过 500 行的大组件
* v-for 没有正确的 key

### 中（考虑修复）

* 模板表达式应该使用 computed
* 不必要的重新渲染（缺少 v-memo）
* 深层响应式过度使用
* 代码中残留 console.log
* 事件名未使用 kebab-case

## 运行的自动化检查

```bash
# Vue SFC type checking
vue-tsc --noEmit

# ESLint with Vue rules
eslint . --ext .vue,.js,.jsx,.ts,.tsx

# Vue-specific lint rules
eslint . --ext .vue --rule 'vue/require-v-for-key: error'

# Check for unused components
npx vue-unused-components

# Bundle analysis (if available)
npm run build -- --analyze
```

## 使用示例

```text
User: /vue-review

Agent:
# Vue Code Review Report

## 已审查的文件
- src/components/UserList.vue (modified)
- src/composables/useAuth.ts (modified)
- src/views/Dashboard.vue (modified)

## 静态分析结果
✓ vue-tsc: 无类型错误
✓ ESLint: 无问题

## 发现的问题

[严重] 直接修改 Props
文件：src/components/UserList.vue:28
问题：子组件直接修改 props
```vue
<script setup>
const props = defineProps(['user'])
// 错误：直接修改
props.user.name = 'New Name'
</script>
```
修复：通过 emit 事件让父组件更新
```vue
<script setup>
const props = defineProps(['user'])
const emit = defineEmits(['update:user'])

// 正确：emit 事件
emit('update:user', { ...props.user, name: 'New Name' })
</script>
```

[高] 解构导致响应性丢失
文件：src/composables/useAuth.ts:15
问题：解构 reactive 对象会丢失响应性
```typescript
// 错误：丢失响应性
const { user, token } = reactive(authState)

// 正确：使用 toRefs
const { user, token } = toRefs(reactive(authState))
```

[高] 缺少事件监听器清理
文件：src/views/Dashboard.vue:42
问题：事件监听器在卸载时未清理
```vue
<script setup>
onMounted(() => {
  window.addEventListener('resize', handleResize)
  // 缺少清理！
})
</script>
```
修复：在 onUnmounted 中添加清理
```vue
<script setup>
onMounted(() => {
  window.addEventListener('resize', handleResize)
})
onUnmounted(() => {
  window.removeEventListener('resize', handleResize)
})
</script>
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

## Vue 3 特定检查

审查 Vue 3 代码时，还需验证：
- `defineProps` 和 `defineEmits` 有正确的 TypeScript 类型
- `ref` vs `reactive` 使用得当
- 解构 reactive 对象时使用 `toRefs`
- `watchEffect` 清理函数已实现
- 异步组件有 Suspense 边界
- `<script setup>` 中没有混用 Options API

## Vue 2 特定检查

审查 Vue 2 代码时，还需验证：
- `data` 是函数，不是对象
- 正确使用 `this.$set` 进行响应式添加
- 没有箭头函数丢失 `this` 上下文
- 过滤器已为 Vue 3 迁移做准备

## 与其他命令的集成

- 先使用 `/vue-test` 确保测试通过
- 如果出现构建错误使用 `/vue-build`
- 提交前使用 `/vue-review`
- 对于非 Vue 特定问题使用 `/code-review`

## 相关

- Agent: `agents/vue-reviewer.md`
- Skills: `skills/vue-patterns/`, `skills/vue-testing/`
