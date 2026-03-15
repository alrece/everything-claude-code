---
paths:
  - "**/*.vue"
  - "**/*.ts"
  - "**/*.tsx"
---
# Vue Security

> This file extends [common/security.md](../common/security.md) with Vue specific content.

## XSS Prevention

### v-html Danger

Never use `v-html` with user-provided content:

```vue
<!-- DANGEROUS: XSS vulnerability -->
<div v-html="userContent"></div>

<!-- SAFE: Auto-escaped -->
<div>{{ userContent }}</div>

<!-- If HTML is required, sanitize first -->
<script setup lang="ts">
import DOMPurify from 'dompurify'

const sanitizedContent = computed(() =>
  DOMPurify.sanitize(userContent.value)
)
</script>

<div v-html="sanitizedContent"></div>
```

### URL Binding

Validate URLs before binding to href or src:

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
  <a :href="safeUrl">Link</a>
</template>
```

## Secret Management

Never expose secrets in client-side code:

```typescript
// WRONG: Exposed in bundle
const apiKey = "sk-proj-xxxxx"
const apiSecret = "secret123"

// CORRECT: Use environment variables at build time
const config = {
  apiBaseUrl: import.meta.env.VITE_API_BASE_URL,
  // API calls go through backend, not directly from client
}

// CORRECT: Backend proxy pattern
async function callApi(endpoint: string) {
  // Let backend handle authentication
  const response = await fetch(`/api/proxy/${endpoint}`)
  return response.json()
}
```

## Props Validation

Always validate props from untrusted sources:

```typescript
import { z } from 'zod'

const UserSchema = z.object({
  id: z.number(),
  name: z.string(),
  email: z.string().email(),
  role: z.enum(['user', 'admin'])
})

// Validate before using
const props = defineProps<{ user: unknown }>()

const validatedUser = computed(() =>
  UserSchema.parse(props.user)
)
```

## Sensitive Data in Templates

Never render sensitive data directly:

```vue
<!-- WRONG: Exposed in HTML -->
<div>User Token: {{ user.token }}</div>
<input :value="user.password" />

<!-- CORRECT: Mask sensitive data -->
<div>User Token: ****{{ token.slice(-4) }}</div>
<input type="password" v-model="password" />
```

## Event Handler Security

Validate data before emitting:

```vue
<script setup lang="ts">
const emit = defineEmits<{
  (e: 'submit', data: FormData): void
}>()

function handleSubmit(formData: unknown) {
  // Validate before emitting
  const validated = FormSchema.parse(formData)
  emit('submit', validated)
}
</script>
```

## Local Storage Security

Never store sensitive data in localStorage:

```typescript
// WRONG: Sensitive data in localStorage
localStorage.setItem('token', userToken)
localStorage.setItem('user', JSON.stringify(user))

// CORRECT: Use secure storage or httpOnly cookies
// Store only non-sensitive preferences
localStorage.setItem('theme', 'dark')
localStorage.setItem('language', 'en')

// For session data, use sessionStorage (cleared on tab close)
sessionStorage.setItem('tempData', JSON.stringify(data))
```

## Third-Party Component Security

Vet third-party components before use:

```typescript
// Check for:
// 1. Active maintenance
// 2. Security advisories
// 3. Bundle size and dependencies
// 4. Proper license

// Use npm audit
// npm audit

// Check dependencies
// npm ls <package-name>
```

## Content Security Policy

Configure CSP headers on your server:

```html
<meta http-equiv="Content-Security-Policy" content="
  default-src 'self';
  script-src 'self' 'unsafe-inline' 'unsafe-eval';
  style-src 'self' 'unsafe-inline';
  img-src 'self' data: https:;
  connect-src 'self' https://api.example.com;
">
```

## Agent Support

- Use **security-reviewer** agent for comprehensive security audits
- Use **vue-reviewer** agent for Vue-specific security patterns

## Reference

See skill: `vue-patterns` for security patterns and best practices.
