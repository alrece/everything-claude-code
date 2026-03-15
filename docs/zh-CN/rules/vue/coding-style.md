---
paths:
  - "**/*.vue"
  - "**/*.ts"
  - "**/*.tsx"
---
# Vue 编码风格

> 此文件扩展了 [common/coding-style.md](../common/coding-style.md) 的 Vue 特定内容。

## 组件结构

使用 `<script setup>` 语法的单文件组件（SFC）：

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

// 响应式状态
const count = ref(0)

// 计算属性
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

## 命名约定

- **组件**：文件和导入中使用 PascalCase（`UserCard.vue`、`<UserCard />`）
- **Props**：模板中使用 kebab-case（`user-name`），脚本中使用 camelCase（`userName`）
- **事件**：使用 kebab-case（`@update-user`、`@item-click`）
- **Composables**：使用 `use` 前缀的 camelCase（`useAuth`、`useFetch`）

## Props 和 Emits

始终为 props 和 emits 定义类型：

```typescript
// 错误：无类型 props
const props = defineProps(['user', 'items'])

// 正确：有类型 props
interface Props {
  user: User
  items: Item[]
  loading?: boolean
}

const props = defineProps<Props>()

// 带默认值
const props = withDefaults(defineProps<Props>(), {
  loading: false
})
```

## 响应式

### ref vs reactive

- 原始类型和单个值使用 `ref`
- 需要深度响应式的对象使用 `reactive`
- 不确定时优先使用 `ref` 保持一致性

```typescript
// 原始类型 - 使用 ref
const count = ref(0)
const name = ref('')

// 对象 - 使用 ref 便于替换
const user = ref<User | null>(null)
user.value = await fetchUser() // 轻松替换

// 对象 - 使用 reactive 便于修改
const state = reactive({
  count: 0,
  name: ''
})
state.count++ // 直接修改
```

### 解构

永远不要直接解构响应式对象 - 使用 `toRefs`：

```typescript
// 错误：丢失响应性
const { count, name } = reactive({ count: 0, name: '' })

// 正确：保持响应性
const state = reactive({ count: 0, name: '' })
const { count, name } = toRefs(state)

// 或用于 props
const { title, items } = toRefs(props)
```

## 计算属性

使用 computed 处理派生状态：

```vue
<script setup lang="ts">
const items = ref<Item[]>([])

// 正确：计算属性处理派生状态
const activeItems = computed(() =>
  items.value.filter(item => item.active)
)

const itemCount = computed(() => items.value.length)

// 错误：方法处理简单派生状态
function getActiveItems() {
  return items.value.filter(item => item.active)
}
</script>
```

## v-for 和 Keys

始终使用稳定的唯一键：

```vue
<!-- 错误：索引作为 key -->
<li v-for="(item, index) in items" :key="index">

<!-- 正确：唯一 id 作为 key -->
<li v-for="item in items" :key="item.id">
```

## v-if 和 v-for

永远不要在同一元素上使用 v-if 和 v-for：

```vue
<!-- 错误：同一元素上使用 v-if 和 v-for -->
<li v-for="item in items" v-if="item.active" :key="item.id">

<!-- 正确：使用计算属性或嵌套 template -->
<template v-for="item in items" :key="item.id">
  <li v-if="item.active">{{ item.name }}</li>
</template>

<!-- 正确：计算属性 -->
<li v-for="item in activeItems" :key="item.id">
```

## 组件大小

保持组件专注且在 500 行以内：

- 将大型组件拆分为更小的组件
- 使用 composables 共享逻辑
- 将复杂模板拆分为子组件

## Console.log

* 生产代码中不要有 `console.log`
* 使用正确的日志记录或在提交前删除
* 参见 hooks 进行自动检测

## 参考

详细的 Vue 模式和最佳实践请参阅 skill: `vue-patterns`。
