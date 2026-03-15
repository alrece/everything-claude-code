---
paths:
  - "**/*.vue"
  - "**/*.test.ts"
  - "**/*.spec.ts"
---
# Vue Testing

> This file extends [common/testing.md](../common/testing.md) with Vue specific content.

## Framework

Use **Vitest** with **Vue Test Utils** for unit and component testing.

```bash
npm install -D vitest @vue/test-utils jsdom
```

## Component Testing

### Basic Mount

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

### Testing Props

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

### Testing Events

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

### Testing v-model

```typescript
it('updates v-model on input', async () => {
  const wrapper = mount(Input, {
    props: { modelValue: 'initial' }
  })

  await wrapper.find('input').setValue('new value')

  expect(wrapper.emitted('update:modelValue')![0]).toEqual(['new value'])
})
```

### Testing Slots

```typescript
it('renders default slot', () => {
  const wrapper = mount(Card, {
    slots: {
      default: 'Card content'
    }
  })

  expect(wrapper.text()).toContain('Card content')
})

it('renders named slots', () => {
  const wrapper = mount(Layout, {
    slots: {
      header: '<h1>Header</h1>',
      default: '<p>Content</p>',
      footer: '<span>Footer</span>'
    }
  })

  expect(wrapper.html()).toContain('<h1>Header</h1>')
  expect(wrapper.html()).toContain('<p>Content</p>')
})
```

### Testing Async Behavior

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

### Testing with Plugins

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

  // Component can access the store
})
```

## Composable Testing

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

## Coverage

```bash
# Run with coverage
vitest run --coverage

# Coverage targets
# - Critical components: 100%
# - Shared components: 90%+
# - General components: 80%+
```

## E2E Testing

Use **Playwright** or **Cypress** for E2E testing:

```typescript
// Playwright example
import { test, expect } from '@playwright/test'

test('user can login', async ({ page }) => {
  await page.goto('/login')
  await page.fill('[name="email"]', 'user@example.com')
  await page.fill('[name="password"]', 'password123')
  await page.click('button[type="submit"]')

  await expect(page).toHaveURL('/dashboard')
})
```

## Best Practices

**DO:**
- Test behavior, not implementation details
- Use `data-testid` for stable selectors
- Mock external dependencies
- Test edge cases (loading, error, empty states)
- Clean up after tests

**DON'T:**
- Test private methods directly
- Rely on CSS classes for assertions
- Test third-party library functionality
- Use `wrapper.vm` to access internal state

## Agent Support

- **vue-test** - TDD workflow for Vue components
- **e2e-runner** - Playwright E2E testing specialist

## Reference

See skill: `vue-testing` for detailed Vue testing patterns and helpers.
