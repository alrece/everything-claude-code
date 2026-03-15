---
description: Enforce TDD workflow for Vue. Write component tests first using Vitest/Jest, then implement. Verify 80%+ coverage with Vue Test Utils.
---

# Vue TDD Command

This command enforces test-driven development methodology for Vue code using Vitest or Jest with Vue Test Utils.

## What This Command Does

1. **Define Component Interface**: Scaffold component with props, emits, slots
2. **Write Component Tests**: Create comprehensive test cases (RED)
3. **Run Tests**: Verify tests fail for the right reason
4. **Implement Component**: Write minimal code to pass (GREEN)
5. **Refactor**: Improve while keeping tests green
6. **Check Coverage**: Ensure 80%+ coverage

## When to Use

Use `/vue-test` when:
- Implementing new Vue components
- Adding test coverage to existing components
- Fixing bugs (write failing test first)
- Building critical business logic
- Learning TDD workflow in Vue

## TDD Cycle

```
RED     → Write failing component test
GREEN   → Implement minimal component to pass
REFACTOR → Improve component, tests stay green
REPEAT  → Next test case
```

## Example Session

````text
User: /vue-test I need a UserCard component that displays user info with an edit button

Agent:
# TDD Session: UserCard Component

## Step 1: Define Component Interface

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
  <!-- To be implemented -->
</template>
```

## Step 2: Write Component Tests (RED)

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

## Step 3: Run Tests - Verify FAIL

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

✓ Tests fail as expected (not implemented).

## Step 4: Implement Minimal Code (GREEN)

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

## Step 5: Run Tests - Verify PASS

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

✓ All tests passing!

## Step 6: Check Coverage

```bash
$ npm run test -- --coverage

PASS src/components/UserCard.test.ts
Coverage: 100% Statements | 100% Branches | 100% Functions | 100% Lines
```

✓ Coverage: 100%

## TDD Complete!
````

## Test Patterns

### Component Mounting

```typescript
// Basic mount
const wrapper = mount(Component, {
  props: { title: 'Hello' }
})

// With slots
const wrapper = mount(Component, {
  slots: {
    default: 'Slot content',
    header: '<h1>Header</h1>'
  }
})

// With global plugins
const wrapper = mount(Component, {
  global: {
    plugins: [router, pinia],
    provide: { theme: darkTheme }
  }
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
```

### Testing Events

```typescript
it('emits close event', async () => {
  const wrapper = mount(Modal)
  await wrapper.find('.close-btn').trigger('click')
  expect(wrapper.emitted('close')).toBeTruthy()
})
```

### Testing Slots

```typescript
it('renders slot content', () => {
  const wrapper = mount(Card, {
    slots: { default: 'Custom content' }
  })
  expect(wrapper.text()).toContain('Custom content')
})
```

### Testing v-model

```typescript
it('updates v-model', async () => {
  const wrapper = mount(Input, {
    props: { modelValue: 'initial' }
  })
  await wrapper.find('input').setValue('new value')
  expect(wrapper.emitted('update:modelValue')![0]).toEqual(['new value'])
})
```

### Testing Composables

```typescript
it('returns reactive data', () => {
  const { result } = renderHook(() => useCounter())
  expect(result.count.value).toBe(0)
  result.increment()
  expect(result.count.value).toBe(1)
})
```

## Coverage Commands

```bash
# Vitest coverage
npm run test -- --coverage

# Jest coverage
npm run test -- --coverage --watchAll=false

# Vue Test Utils with coverage
vitest run --coverage
```

## Coverage Targets

| Code Type | Target |
|-----------|--------|
| Critical components | 100% |
| Shared components | 90%+ |
| General components | 80%+ |
| Generated code | Exclude |

## TDD Best Practices

**DO:**
- Write test FIRST, before any implementation
- Test behavior, not implementation details
- Test user interactions (click, input, etc.)
- Test edge cases (empty, loading, error states)
- Use `data-testid` for stable selectors

**DON'T:**
- Test private methods directly
- Use `wrapper.vm` to access internal state
- Rely on CSS classes for assertions
- Test third-party library behavior
- Skip the RED phase

## Related Commands

- `/vue-build` - Fix build errors
- `/vue-review` - Review code after implementation
- `/verify` - Run full verification loop

## Related

- Skill: `skills/vue-testing/`
- Skill: `skills/tdd-workflow/`
