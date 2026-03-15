---
paths:
  - "**/*.vue"
  - "**/*.ts"
  - "**/*.tsx"
---
# Vue 模式

> 此文件扩展了 [common/patterns.md](../common/patterns.md) 的 Vue 特定内容。

## Composables

使用 composables 提取和复用有状态逻辑：

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

## 双向绑定的 Props

使用 `defineModel` 支持 v-model：

```vue
<script setup lang="ts">
// 单个 v-model
const modelValue = defineModel<string>()

// 带选项
const title = defineModel<string>('title', { default: '' })

// 多个 v-model
const firstName = defineModel<string>('firstName')
const lastName = defineModel<string>('lastName')
</script>

<template>
  <input v-model="modelValue" />
</template>
```

## Provide/Inject 模式

用于深层组件通信：

```typescript
// 父组件：Provide
import { provide, ref } from 'vue'

const theme = ref('dark')
provide('theme', theme)

// 或使用 InjectionKey 实现类型安全
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
<!-- 子组件：Inject -->
<script setup lang="ts">
import { inject } from 'vue'
import { ThemeKey } from './keys'

const { theme, toggle } = inject(ThemeKey)!
</script>
```

## 异步 Setup 模式

在 setup 中处理异步操作：

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
  <div v-if="loading">加载中...</div>
  <div v-else-if="error">错误：{{ error.message }}</div>
  <div v-else>{{ data }}</div>
</template>
```

## Pinia Store 模式

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

## 组件中的错误处理

```vue
<script setup lang="ts">
import { ref, onErrorCaptured } from 'vue'

const error = ref<Error | null>(null)

// 捕获子组件的错误
onErrorCaptured((err) => {
  error.value = err
  return false // 阻止错误向上传播
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
    <!-- 表单内容 -->
  </form>
</template>
```

## 懒加载组件

```typescript
// 使用 defineAsyncComponent 懒加载
import { defineAsyncComponent } from 'vue'

const HeavyComponent = defineAsyncComponent(() =>
  import('./HeavyComponent.vue')
)

// 带加载和错误状态
const AsyncComponent = defineAsyncComponent({
  loader: () => import('./AsyncComponent.vue'),
  loadingComponent: LoadingSpinner,
  errorComponent: ErrorDisplay,
  delay: 200,
  timeout: 3000
})
```

## 插槽模式

### 默认插槽

```vue
<!-- Card.vue -->
<template>
  <div class="card">
    <slot />
  </div>
</template>

<!-- 使用 -->
<Card>
  <p>卡片内容</p>
</Card>
```

### 具名插槽

```vue
<!-- Layout.vue -->
<template>
  <div class="layout">
    <header><slot name="header" /></header>
    <main><slot /></main>
    <footer><slot name="footer" /></footer>
  </div>
</template>

<!-- 使用 -->
<Layout>
  <template #header>头部</template>
  主要内容
  <template #footer>页脚</template>
</Layout>
```

### 作用域插槽

```vue
<!-- List.vue -->
<script setup lang="ts">
const items = defineProps<{ items: Item[] }>()
</script>

<template>
  <ul>
    <li v-for="(item, index) in items" :key="item.id">
      <slot :item="item" :index="index" />
    </li>
  </ul>
</template>

<!-- 使用 -->
<List :items="users">
  <template #default="{ item, index }">
    {{ index }}: {{ item.name }}
  </template>
</List>
```

## 参考

详细的 Vue 模式（包括状态管理、composables 和组件设计）请参阅 skill: `vue-patterns`。
