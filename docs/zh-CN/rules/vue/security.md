---
paths:
  - "**/*.vue"
  - "**/*.ts"
  - "**/*.tsx"
---
# Vue 安全

> 此文件扩展了 [common/security.md](../common/security.md) 的 Vue 特定内容。

## XSS 防护

### v-html 危险

永远不要将 `v-html` 用于用户提供的内容：

```vue
<!-- 危险：XSS 漏洞 -->
<div v-html="userContent"></div>

<!-- 安全：自动转义 -->
<div>{{ userContent }}</div>

<!-- 如果必须使用 HTML，先进行清理 -->
<script setup lang="ts">
import DOMPurify from 'dompurify'

const sanitizedContent = computed(() =>
  DOMPurify.sanitize(userContent.value)
)
</script>

<div v-html="sanitizedContent"></div>
```

### URL 绑定

在绑定到 href 或 src 之前验证 URL：

```vue
<script setup lang="ts">
function isSafeUrl(url: string): boolean {
  try {
    const parsed = new URL(url, window.location.origin)
    return ['http:', 'https:'].includes(parsed.protocol)
  } catch {
    return false
  }
}

const safeUrl = computed(() =>
  isSafeUrl(userUrl.value) ? userUrl.value : '#'
)
</script>

<template>
  <a :href="safeUrl">链接</a>
</template>
```

## 密钥管理

永远不要在客户端代码中暴露密钥：

```typescript
// 错误：在打包文件中暴露
const apiKey = "sk-proj-xxxxx"
const apiSecret = "secret123"

// 正确：在构建时使用环境变量
const config = {
  apiBaseUrl: import.meta.env.VITE_API_BASE_URL,
  // API 调用通过后端，而不是直接从客户端
}

// 正确：后端代理模式
async function callApi(endpoint: string) {
  // 让后端处理身份验证
  const response = await fetch(`/api/proxy/${endpoint}`)
  return response.json()
}
```

## Props 验证

始终验证来自不可信来源的 props：

```typescript
import { z } from 'zod'

const UserSchema = z.object({
  id: z.number(),
  name: z.string(),
  email: z.string().email(),
  role: z.enum(['user', 'admin'])
})

// 使用前验证
const props = defineProps<{ user: unknown }>()

const validatedUser = computed(() =>
  UserSchema.parse(props.user)
)
```

## 模板中的敏感数据

永远不要直接渲染敏感数据：

```vue
<!-- 错误：在 HTML 中暴露 -->
<div>用户令牌：{{ user.token }}</div>
<input :value="user.password" />

<!-- 正确：遮蔽敏感数据 -->
<div>用户令牌：****{{ token.slice(-4) }}</div>
<input type="password" v-model="password" />
```

## 事件处理器安全

在触发事件前验证数据：

```vue
<script setup lang="ts">
const emit = defineEmits<{
  (e: 'submit', data: FormData): void
}>()

function handleSubmit(formData: unknown) {
  // 触发前验证
  const validated = FormSchema.parse(formData)
  emit('submit', validated)
}
</script>
```

## 本地存储安全

永远不要在 localStorage 中存储敏感数据：

```typescript
// 错误：localStorage 中存储敏感数据
localStorage.setItem('token', userToken)
localStorage.setItem('user', JSON.stringify(user))

// 正确：使用安全存储或 httpOnly cookies
// 只存储非敏感的偏好设置
localStorage.setItem('theme', 'dark')
localStorage.setItem('language', 'zh-CN')

// 对于会话数据，使用 sessionStorage（关闭标签页时清除）
sessionStorage.setItem('tempData', JSON.stringify(data))
```

## 第三方组件安全

使用前审查第三方组件：

```typescript
// 检查：
// 1. 积极维护
// 2. 安全公告
// 3. 包大小和依赖
// 4. 适当的许可证

// 使用 npm audit
// npm audit

// 检查依赖
// npm ls <package-name>
```

## 内容安全策略

在服务器上配置 CSP 头：

```html
<meta http-equiv="Content-Security-Policy" content="
  default-src 'self';
  script-src 'self' 'unsafe-inline' 'unsafe-eval';
  style-src 'self' 'unsafe-inline';
  img-src 'self' data: https:;
  connect-src 'self' https://api.example.com;
">
```

## Agent 支持

* 使用 **security-reviewer** agent 进行全面的安全审计
* 使用 **vue-reviewer** agent 处理 Vue 特定的安全模式

## 参考

安全模式和最佳实践请参阅 skill: `vue-patterns`。
