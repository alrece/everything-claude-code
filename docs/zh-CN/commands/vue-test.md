---
description: 为Vue强制执行TDD工作流程。使用Vitest/Jest首先编写组件测试，然后实现。使用Vue Test Utils验证80%以上的覆盖率。
---

# Vue TDD 命令

此命令使用 Vitest 或 Jest 配合 Vue Test Utils，为 Vue 代码强制执行测试驱动开发方法。

## 此命令的作用

1. **定义组件接口**：搭建组件的 props、emits、slots
2. **编写组件测试**：创建全面的测试用例（RED 阶段）
3. **运行测试**：验证测试因正确原因而失败
4. **实现组件**：编写最少的代码以通过测试（GREEN 阶段）
5. **重构**：改进代码，同时保持测试通过
6. **检查覆盖率**：确保 80% 以上的覆盖率

## 何时使用

在以下情况下使用 `/vue-test`：

* 实现新的 Vue 组件时
* 为现有组件添加测试覆盖率时
* 修复 bug 时（先编写失败的测试）
* 构建关键业务逻辑时
* 学习 Vue 中的 TDD 工作流程时

## TDD 循环

```
RED     → 编写失败的组件测试
GREEN   → 实现最少的代码以通过测试
REFACTOR → 改进组件，测试保持通过
REPEAT  → 下一个测试用例
```

## 示例会话

````text
User: /vue-test I need a UserCard component that displays user info with an edit button

Agent:
# TDD Session: UserCard Component

## 步骤 1：定义组件接口

```vue
<!-- src/components/UserCard.vue -->
<script setup lang="ts">
interface User {
  id: number
  name: string
  email: string
  avatar?: string
}

defineProps<{
  user: User
}>()

defineEmits<{
  (e: 'edit', id: number): void
}>()
</script>

<template>
  <!-- 待实现 -->
</template>
```

## 步骤 2：编写组件测试（RED 阶段）

```typescript
// src/components/UserCard.test.ts
import { describe, it, expect, vi } from 'vitest'
import { mount } from '@vue/test-utils'
import UserCard from './UserCard.vue'

describe('UserCard', () => {
  const mockUser = {
    id: 1,
    name: 'John Doe',
    email: 'john@example.com',
    avatar: 'https://example.com/avatar.jpg'
  }

  it('renders user name', () => {
    const wrapper = mount(UserCard, {
      props: { user: mockUser }
    })
    expect(wrapper.text()).toContain('John Doe')
  })

  it('renders user email', () => {
    const wrapper = mount(UserCard, {
      props: { user: mockUser }
    })
    expect(wrapper.text()).toContain('john@example.com')
  })

  it('renders avatar when provided', () => {
    const wrapper = mount(UserCard, {
      props: { user: mockUser }
    })
    expect(wrapper.find('img').attributes('src')).toBe(mockUser.avatar)
  })

  it('shows placeholder when no avatar', () => {
    const wrapper = mount(UserCard, {
      props: { user: { ...mockUser, avatar: undefined } }
    })
    expect(wrapper.find('.avatar-placeholder').exists()).toBe(true)
  })

  it('emits edit event when edit button clicked', async () => {
    const wrapper = mount(UserCard, {
      props: { user: mockUser }
    })
    await wrapper.find('.edit-button').trigger('click')
    expect(wrapper.emitted('edit')).toBeTruthy()
    expect(wrapper.emitted('edit')![0]).toEqual([1])
  })
})
```

## 步骤 3：运行测试 - 验证失败

```bash
$ npm run test

FAIL src/components/UserCard.test.ts
  ✗ UserCard > renders user name
    AssertionError: expected '' to contain 'John Doe'
  ✗ UserCard > renders user email
    AssertionError: expected '' to contain 'john@example.com'
  ✗ UserCard > emits edit event when edit button clicked
    TypeError: Cannot read properties of undefined (reading 'trigger')

FAIL 5/5 tests
```

✓ 测试按预期失败（未实现）。

## 步骤 4：实现最少的代码（GREEN 阶段）

```vue
<!-- src/components/UserCard.vue -->
<script setup lang="ts">
interface User {
  id: number
  name: string
  email: string
  avatar?: string
}

const props = defineProps<{
  user: User
}>()

const emit = defineEmits<{
  (e: 'edit', id: number): void
}>()

const handleEdit = () => {
  emit('edit', props.user.id)
}
</script>

<template>
  <div class="user-card">
    <img
      v-if="user.avatar"
      :src="user.avatar"
      :alt="user.name"
      class="avatar"
    />
    <div v-else class="avatar-placeholder">
      {{ user.name.charAt(0).toUpperCase() }}
    </div>

    <div class="user-info">
      <h3 class="name">{{ user.name }}</h3>
      <p class="email">{{ user.email }}</p>
    </div>

    <button class="edit-button" @click="handleEdit">
      Edit
    </button>
  </div>
</template>
```

## 步骤 5：运行测试 - 验证通过

```bash
$ npm run test

PASS src/components/UserCard.test.ts
  ✓ UserCard > renders user name
  ✓ UserCard > renders user email
  ✓ UserCard > renders avatar when provided
  ✓ UserCard > shows placeholder when no avatar
  ✓ UserCard > emits edit event when edit button clicked

PASS 5/5 tests
```

✓ 所有测试通过！

## 步骤 6：检查覆盖率

```bash
$ npm run test -- --coverage

PASS src/components/UserCard.test.ts
Coverage: 100% Statements | 100% Branches | 100% Functions | 100% Lines
```

✓ 覆盖率：100%

## TDD 完成！
````

## 测试模式

### 组件挂载

```typescript
// 基本挂载
const wrapper = mount(Component, {
  props: { title: 'Hello' }
})

// 带插槽
const wrapper = mount(Component, {
  slots: {
    default: 'Slot content',
    header: '<h1>Header</h1>'
  }
})

// 带全局插件
const wrapper = mount(Component, {
  global: {
    plugins: [router, pinia],
    provide: { theme: darkTheme }
  }
})
```

### 测试 Props

```typescript
it('displays title from prop', () => {
  const wrapper = mount(Card, {
    props: { title: 'Test Title' }
  })
  expect(wrapper.find('h2').text()).toBe('Test Title')
})
```

### 测试事件

```typescript
it('emits close event', async () => {
  const wrapper = mount(Modal)
  await wrapper.find('.close-btn').trigger('click')
  expect(wrapper.emitted('close')).toBeTruthy()
})
```

### 测试插槽

```typescript
it('renders slot content', () => {
  const wrapper = mount(Card, {
    slots: { default: 'Custom content' }
  })
  expect(wrapper.text()).toContain('Custom content')
})
```

### 测试 v-model

```typescript
it('updates v-model', async () => {
  const wrapper = mount(Input, {
    props: { modelValue: 'initial' }
  })
  await wrapper.find('input').setValue('new value')
  expect(wrapper.emitted('update:modelValue')![0]).toEqual(['new value'])
})
```

### 测试 Composables

```typescript
it('returns reactive data', () => {
  const { result } = renderHook(() => useCounter())
  expect(result.count.value).toBe(0)
  result.increment()
  expect(result.count.value).toBe(1)
})
```

## 覆盖率命令

```bash
# Vitest coverage
npm run test -- --coverage

# Jest coverage
npm run test -- --coverage --watchAll=false

# Vue Test Utils with coverage
vitest run --coverage
```

## 覆盖率目标

| 代码类型 | 目标 |
|-----------|--------|
| 关键组件 | 100% |
| 共享组件 | 90%+ |
| 通用组件 | 80%+ |
| 生成的代码 | 排除 |

## TDD 最佳实践

**应该做：**

* 先编写测试，再编写任何实现
* 测试行为，而非实现细节
* 测试用户交互（点击、输入等）
* 包含边界情况（空值、加载、错误状态）
* 使用 `data-testid` 作为稳定的选择器

**不应该做：**

* 直接测试私有方法
* 使用 `wrapper.vm` 访问内部状态
* 依赖 CSS 类进行断言
* 测试第三方库行为
* 跳过 RED 阶段

## 相关命令

* `/vue-build` - 修复构建错误
* `/vue-review` - 在实现后审查代码
* `/verify` - 运行完整的验证循环

## 相关

* 技能：`skills/vue-testing/`
* 技能：`skills/tdd-workflow/`
