---
paths:
  - "**/*.vue"
  - "**/*.ts"
  - "**/*.tsx"
---
# Vue Patterns

> This file extends [common/patterns.md](../common/patterns.md) with Vue specific content.

## Composables

Extract and reuse stateful logic with composables:

```typescript
// composables/useAuth.ts
import { ref, computed } from 'vue'

const user = ref<User | null>(null)
const loading = ref(false)

export function useAuth() {
  const isAuthenticated = computed(() => !!user.value)

  async function login(credentials: Credentials) {
    loading.value = true
    try {
      user.value = await authApi.login(credentials)
    } finally {
      loading.value = false
    }
  }

  function logout() {
    user.value = null
    authApi.logout()
  }

  return {
    user,
    loading,
    isAuthenticated,
    login,
    logout
  }
}
```

## Props with Two-Way Binding

Use `defineModel` for v-model support:

```vue
<script setup lang="ts">
// Single v-model
const modelValue = defineModel<string>()

// With options
const title = defineModel<string>('title', { default: '' })

// Multiple v-models
const firstName = defineModel<string>('firstName')
const lastName = defineModel<string>('lastName')
</script>

<template>
  <input v-model="modelValue" />
</template>
```

## Provide/Inject Pattern

For deep component communication:

```typescript
// Parent: Provide
import { provide, ref } from 'vue'

const theme = ref('dark')
provide('theme', theme)

// Or with InjectionKey for type safety
import type { InjectionKey } from 'vue'

interface ThemeContext {
  theme: Ref<string>
  toggle: () => void
}

const ThemeKey: InjectionKey<ThemeContext> = Symbol('theme')
provide(ThemeKey, {
  theme,
  toggle: () => theme.value = theme.value === 'dark' ? 'light' : 'dark'
})
```

```vue
<!-- Child: Inject -->
<script setup lang="ts">
import { inject } from 'vue'
import { ThemeKey } from './keys'

const { theme, toggle } = inject(ThemeKey)!
</script>
```

## Async Setup Pattern

Handle async operations in setup:

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue'

const data = ref<Data | null>(null)
const error = ref<Error | null>(null)
const loading = ref(true)

onMounted(async () => {
  try {
    data.value = await fetchData()
  } catch (e) {
    error.value = e instanceof Error ? e : new Error(String(e))
  } finally {
    loading.value = false
  }
})
</script>

<template>
  <div v-if="loading">Loading...</div>
  <div v-else-if="error">Error: {{ error.message }}</div>
  <div v-else>{{ data }}</div>
</template>
```

## Store Pattern with Pinia

```typescript
// stores/user.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

export const useUserStore = defineStore('user', () => {
  // State
  const users = ref<User[]>([])
  const currentUserId = ref<string | null>(null)

  // Getters
  const currentUser = computed(() =>
    users.value.find(u => u.id === currentUserId.value)
  )

  // Actions
  async function fetchUsers() {
    users.value = await userApi.getAll()
  }

  function setUser(userId: string) {
    currentUserId.value = userId
  }

  return {
    users,
    currentUserId,
    currentUser,
    fetchUsers,
    setUser
  }
})
```

## Error Handling in Components

```vue
<script setup lang="ts">
import { ref, onErrorCaptured } from 'vue'

const error = ref<Error | null>(null)

// Capture errors from child components
onErrorCaptured((err) => {
  error.value = err
  return false // Prevent error from propagating
})

async function handleSubmit() {
  try {
    await saveData(formData)
  } catch (err) {
    error.value = err instanceof Error ? err : new Error(String(err))
  }
}
</script>

<template>
  <ErrorBoundary v-if="error" :error="error" />
  <form v-else @submit.prevent="handleSubmit">
    <!-- form content -->
  </form>
</template>
```

## Lazy Loading Components

```typescript
// Lazy load with defineAsyncComponent
import { defineAsyncComponent } from 'vue'

const HeavyComponent = defineAsyncComponent(() =>
  import('./HeavyComponent.vue')
)

// With loading and error states
const AsyncComponent = defineAsyncComponent({
  loader: () => import('./AsyncComponent.vue'),
  loadingComponent: LoadingSpinner,
  errorComponent: ErrorDisplay,
  delay: 200,
  timeout: 3000
})
```

## Slot Patterns

### Default Slot

```vue
<!-- Card.vue -->
<template>
  <div class="card">
    <slot />
  </div>
</template>

<!-- Usage -->
<Card>
  <p>Card content</p>
</Card>
```

### Named Slots

```vue
<!-- Layout.vue -->
<template>
  <div class="layout">
    <header><slot name="header" /></header>
    <main><slot /></main>
    <footer><slot name="footer" /></footer>
  </div>
</template>

<!-- Usage -->
<Layout>
  <template #header>Header</template>
  Main content
  <template #footer>Footer</template>
</Layout>
```

### Scoped Slots

```vue
<!-- List.vue -->
<script setup lang="ts">
const items = defineProps<{ items: Item[] }>()
</script>

<template>
  <ul>
    <li v-for="item in items" :key="item.id">
      <slot :item="item" :index="index" />
    </li>
  </ul>
</template>

<!-- Usage -->
<List :items="users">
  <template #default="{ item, index }">
    {{ index }}: {{ item.name }}
  </template>
</List>
```

## Reference

See skill: `vue-patterns` for comprehensive Vue patterns including state management, composables, and component design.
