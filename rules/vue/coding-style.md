---
paths:
  - "**/*.vue"
  - "**/*.ts"
  - "**/*.tsx"
---
# Vue Coding Style

> This file extends [common/coding-style.md](../common/coding-style.md) with Vue specific content.

## Component Structure

Use Single File Components (SFC) with `<script setup>` syntax:

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

// Props
interface Props {
  title: string
  items?: Item[]
}

const props = withDefaults(defineProps<Props>(), {
  items: () => []
})

// Emits
const emit = defineEmits<{
  (e: 'update', value: string): void
  (e: 'delete', id: number): void
}>()

// Reactive state
const count = ref(0)

// Computed
const doubled = computed(() => count.value * 2)
</script>

<template>
  <div class="component">
    <h2>{{ title }}</h2>
    <ul>
      <li v-for="item in items" :key="item.id">{{ item.name }}</li>
    </ul>
  </div>
</template>

<style scoped>
.component {
  padding: 1rem;
}
</style>
```

## Naming Conventions

- **Components**: PascalCase in files and imports (`UserCard.vue`, `<UserCard />`)
- **Props**: kebab-case in templates (`user-name`), camelCase in script (`userName`)
- **Events**: kebab-case (`@update-user`, `@item-click`)
- **Composables**: camelCase with `use` prefix (`useAuth`, `useFetch`)

## Props and Emits

Always define types for props and emits:

```typescript
// WRONG: Untyped props
const props = defineProps(['user', 'items'])

// CORRECT: Typed props
interface Props {
  user: User
  items: Item[]
  loading?: boolean
}

const props = defineProps<Props>()

// With defaults
const props = withDefaults(defineProps<Props>(), {
  loading: false
})
```

## Reactivity

### ref vs reactive

- Use `ref` for primitives and single values
- Use `reactive` for objects that need deep reactivity
- Prefer `ref` for consistency when in doubt

```typescript
// Primitives - use ref
const count = ref(0)
const name = ref('')

// Objects - use ref for replacement
const user = ref<User | null>(null)
user.value = await fetchUser() // Easy replacement

// Objects - use reactive for mutations
const state = reactive({
  count: 0,
  name: ''
})
state.count++ // Direct mutation
```

### Destructuring

Never destructure reactive objects directly - use `toRefs`:

```typescript
// WRONG: Loses reactivity
const { count, name } = reactive({ count: 0, name: '' })

// CORRECT: Preserves reactivity
const state = reactive({ count: 0, name: '' })
const { count, name } = toRefs(state)

// Or with props
const { title, items } = toRefs(props)
```

## Computed Properties

Use computed for derived state:

```vue
<script setup lang="ts">
const items = ref<Item[]>([])

// CORRECT: Computed for derived state
const activeItems = computed(() =>
  items.value.filter(item => item.active)
)

const itemCount = computed(() => items.value.length)

// WRONG: Method for simple derived state
function getActiveItems() {
  return items.value.filter(item => item.active)
}
</script>
```

## v-for and Keys

Always use stable, unique keys:

```vue
<!-- WRONG: Index as key -->
<li v-for="(item, index) in items" :key="index">

<!-- CORRECT: Unique id as key -->
<li v-for="item in items" :key="item.id">
```

## v-if and v-for

Never use v-if and v-for on the same element:

```vue
<!-- WRONG: v-if and v-for on same element -->
<li v-for="item in items" v-if="item.active" :key="item.id">

<!-- CORRECT: Use computed or nested template -->
<template v-for="item in items" :key="item.id">
  <li v-if="item.active">{{ item.name }}</li>
</template>

<!-- CORRECT: Computed property -->
<li v-for="item in activeItems" :key="item.id">
```

## Component Size

Keep components focused and under 500 lines:

- Extract large components into smaller ones
- Use composables for shared logic
- Split complex templates into child components

## Console.log

- No `console.log` in production code
- Use proper logging or remove before commit
- See hooks for automatic detection

## Reference

See skill: `vue-patterns` for comprehensive Vue patterns and best practices.
