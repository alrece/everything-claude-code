---
paths:
  - "**/*.vue"
  - "**/*.test.ts"
  - "**/*.spec.ts"
---
# Vue 测试

> 此文件扩展了 [common/testing.md](../common/testing.md) 的 Vue 特定内容。

## 框架

使用 **Vitest** 配合 **Vue Test Utils** 进行单元和组件测试。

```bash
npm install -D vitest @vue/test-utils jsdom
```

## 组件测试

### 基本挂载

```typescript
import { mount } from '@vue/test-utils'
import UserCard from './UserCard.vue'

describe('UserCard', () => {
  it('renders user name', () => {
    const wrapper = mount(UserCard, {
      props: {
        user: { id: 1, name: 'John Doe', email: 'john@example.com' }
      }
    })

    expect(wrapper.text()).toContain('John Doe')
  })
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

it('applies variant class', () => {
  const wrapper = mount(Card, {
    props: { variant: 'primary' }
  })

  expect(wrapper.classes()).toContain('card--primary')
})
```

### 测试事件

```typescript
it('emits close event when button clicked', async () => {
  const wrapper = mount(Modal)

  await wrapper.find('.close-btn').trigger('click')

  expect(wrapper.emitted('close')).toBeTruthy()
  expect(wrapper.emitted('close')).toHaveLength(1)
})

it('emits event with payload', async () => {
  const wrapper = mount(UserForm)

  await wrapper.find('form').trigger('submit')

  expect(wrapper.emitted('submit')![0]).toEqual([{
    name: 'John',
    email: 'john@example.com'
  }])
})
```

### 测试 v-model

```typescript
it('updates v-model on input', async () => {
  const wrapper = mount(Input, {
    props: { modelValue: 'initial' }
  })

  await wrapper.find('input').setValue('new value')

  expect(wrapper.emitted('update:modelValue')![0]).toEqual(['new value'])
})
```

### 测试插槽

```typescript
it('renders default slot', () => {
  const wrapper = mount(Card, {
    slots: {
      default: '卡片内容'
    }
  })

  expect(wrapper.text()).toContain('卡片内容')
})

it('renders named slots', () => {
  const wrapper = mount(Layout, {
    slots: {
      header: '<h1>头部</h1>',
      default: '<p>内容</p>',
      footer: '<span>页脚</span>'
    }
  })

  expect(wrapper.html()).toContain('<h1>头部</h1>')
  expect(wrapper.html()).toContain('<p>内容</p>')
})
```

### 测试异步行为

```typescript
it('loads data on mount', async () => {
  const wrapper = mount(UserList, {
    global: {
      mocks: {
        $fetch: vi.fn().mockResolvedValue([
          { id: 1, name: 'User 1' },
          { id: 2, name: 'User 2' }
        ])
      }
    }
  })

  expect(wrapper.find('.loading').exists()).toBe(true)

  await flushPromises()

  expect(wrapper.findAll('.user-item')).toHaveLength(2)
})
```

### 测试带插件的组件

```typescript
import { createTestingPinia } from '@pinia/testing'

it('uses store correctly', () => {
  const wrapper = mount(Component, {
    global: {
      plugins: [
        createTestingPinia({
          initialState: {
            user: { name: 'Test User' }
          }
        })
      ]
    }
  })

  // 组件可以访问 store
})
```

## Composable 测试

```typescript
import { renderHook } from '@testing-library/vue'

describe('useCounter', () => {
  it('increments count', () => {
    const { result } = renderHook(() => useCounter())

    expect(result.count.value).toBe(0)
    result.increment()
    expect(result.count.value).toBe(1)
  })
})
```

## 覆盖率

```bash
# 带覆盖率运行
vitest run --coverage

# 覆盖率目标
# - 关键组件：100%
# - 共享组件：90%+
# - 通用组件：80%+
```

## E2E 测试

使用 **Playwright** 或 **Cypress** 进行 E2E 测试：

```typescript
// Playwright 示例
import { test, expect } from '@playwright/test'

test('用户可以登录', async ({ page }) => {
  await page.goto('/login')
  await page.fill('[name="email"]', 'user@example.com')
  await page.fill('[name="password"]', 'password123')
  await page.click('button[type="submit"]')

  await expect(page).toHaveURL('/dashboard')
})
```

## 最佳实践

**应该做：**
* 测试行为，而非实现细节
* 使用 `data-testid` 作为稳定的选择器
* 模拟外部依赖
* 测试边界情况（加载、错误、空状态）
* 测试后清理

**不应该做：**
* 直接测试私有方法
* 依赖 CSS 类进行断言
* 测试第三方库功能
* 使用 `wrapper.vm` 访问内部状态

## Agent 支持

* **vue-test** - Vue 组件的 TDD 工作流程
* **e2e-runner** - Playwright E2E 测试专家

## 参考

详细的 Vue 测试模式和辅助函数请参阅 skill: `vue-testing`。
