---
description: Comprehensive Vue code review for reactivity patterns, component design, Composition API best practices, and performance. Invokes the vue-reviewer agent.
---

# Vue Code Review

This command invokes the **vue-reviewer** agent for comprehensive Vue-specific code review.

## What This Command Does

1. **Identify Vue Changes**: Find modified `.vue`, `.ts`, `.js` files via `git diff`
2. **Run Static Analysis**: Execute `vue-tsc`, ESLint with Vue rules
3. **Reactivity Review**: Check for reactivity pitfalls, ref/reactive misuse
4. **Component Review**: Analyze props, emits, slots, and component structure
5. **Performance Check**: Identify unnecessary re-renders, memory leaks
6. **Generate Report**: Categorize issues by severity

## When to Use

Use `/vue-review` when:
- After writing or modifying Vue components
- Before committing Vue changes
- Reviewing pull requests with Vue code
- Onboarding to a new Vue codebase
- Learning Vue best practices

## Review Categories

### CRITICAL (Must Fix)
- Props mutation from child components
- Memory leaks from uncleared timers/listeners
- XSS vulnerabilities with v-html
- Circular component dependencies
- Hardcoded credentials in templates

### HIGH (Should Fix)
- Reactivity loss from destructuring
- Missing type definitions for props/emit
- watchEffect without cleanup
- Large components over 500 lines
- v-for without proper keys

### MEDIUM (Consider)
- Missing computed for template expressions
- Unnecessary re-renders (missing v-memo)
- Deep reactivity overuse
- Console.log statements left in code
- Non-kebab-case event names

## Automated Checks Run

```bash
# Vue SFC type checking
vue-tsc --noEmit

# ESLint with Vue rules
eslint . --ext .vue,.js,.jsx,.ts,.tsx

# Vue-specific lint rules
eslint . --ext .vue --rule 'vue/require-v-for-key: error'

# Check for unused components
npx vue-unused-components

# Bundle analysis (if available)
npm run build -- --analyze
```

## Example Usage

```text
User: /vue-review

Agent:
# Vue Code Review Report

## Files Reviewed
- src/components/UserList.vue (modified)
- src/composables/useAuth.ts (modified)
- src/views/Dashboard.vue (modified)

## Static Analysis Results
✓ vue-tsc: No type errors
✓ ESLint: No issues

## Issues Found

[CRITICAL] Direct Props Mutation
File: src/components/UserList.vue:28
Issue: Child component directly mutating props
```vue
<script setup>
const props = defineProps(['user'])
// BAD: Direct mutation
props.user.name = 'New Name'
</script>
```
Fix: Emit event for parent to update
```vue
<script setup>
const props = defineProps(['user'])
const emit = defineEmits(['update:user'])

// GOOD: Emit event
emit('update:user', { ...props.user, name: 'New Name' })
</script>
```

[HIGH] Reactivity Lost via Destructuring
File: src/composables/useAuth.ts:15
Issue: Destructuring reactive object loses reactivity
```typescript
// BAD: Loses reactivity
const { user, token } = reactive(authState)

// GOOD: Use toRefs
const { user, token } = toRefs(reactive(authState))
```

[HIGH] Missing Event Listener Cleanup
File: src/views/Dashboard.vue:42
Issue: Event listener not cleaned up on unmount
```vue
<script setup>
onMounted(() => {
  window.addEventListener('resize', handleResize)
  // Missing cleanup!
})
</script>
```
Fix: Add cleanup in onUnmounted
```vue
<script setup>
onMounted(() => {
  window.addEventListener('resize', handleResize)
})
onUnmounted(() => {
  window.removeEventListener('resize', handleResize)
})
</script>
```

## Summary
- CRITICAL: 1
- HIGH: 2
- MEDIUM: 0

Recommendation: ❌ Block merge until CRITICAL issue is fixed
```

## Approval Criteria

| Status | Condition |
|--------|-----------|
| ✅ Approve | No CRITICAL or HIGH issues |
| ⚠️ Warning | Only MEDIUM issues (merge with caution) |
| ❌ Block | CRITICAL or HIGH issues found |

## Vue 3 Specific Checks

When reviewing Vue 3 code, also verify:
- `defineProps` and `defineEmits` have proper TypeScript types
- `ref` vs `reactive` used appropriately
- `toRefs` used when destructuring reactive objects
- `watchEffect` cleanup functions implemented
- Suspense boundaries for async components
- No Options API mixing in `<script setup>`

## Vue 2 Specific Checks

When reviewing Vue 2 code, also verify:
- `data` is a function, not an object
- Proper use of `this.$set` for reactive additions
- No arrow functions losing `this` context
- Filters prepared for Vue 3 migration

## Integration with Other Commands

- Use `/vue-test` first to ensure tests pass
- Use `/vue-build` if build errors occur
- Use `/vue-review` before committing
- Use `/code-review` for non-Vue specific concerns

## Related

- Agent: `agents/vue-reviewer.md`
- Skills: `skills/vue-patterns/`, `skills/vue-testing/`
