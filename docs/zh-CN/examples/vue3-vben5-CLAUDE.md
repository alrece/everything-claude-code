# Vue3 + Vben5 管理后台 — 项目 CLAUDE.md

> 一个使用 Vue3、Vben5 (Ant Design Vue / Element Plus)、TypeScript 实现的企业级管理后台真实示例。
> 将此文件复制到您的项目根目录，并根据您的业务进行自定义。

## 项目概述

**技术栈:** Vue 3.5+, Vite 7+, Vben Admin 5.0, Ant Design Vue 4.x | Element Plus 2.x, TypeScript 5.x, Pinia, Vue Router 4, Tailwind CSS, Axios, Vee-Validate + Zod

**架构:** 采用领域驱动设计的前端架构，清晰的分层结构：视图层、组件层、服务层、状态管理层。基于 Vben5 的最佳实践，支持多 UI 框架切换。

## 关键规则

### Vue 规范

* 始终使用 `<script setup lang="ts">` 语法的单文件组件
* Props 和 Emits 必须定义类型，使用 `defineProps<T>()` 和 `defineEmits<T>()`
* 响应式数据使用 `ref` (原始类型) 或 `reactive` (对象)
* 组件命名使用 PascalCase，文件命名使用 kebab-case
* Composables 使用 `use` 前缀命名 (如 `useAuth`, `usePermission`)

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

// Props 类型定义
interface Props {
  userId: string
  editable?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  editable: false
})

// Emits 类型定义
const emit = defineEmits<{
  (e: 'update', value: User): void
  (e: 'delete', id: string): void
}>()

// 响应式状态
const loading = ref(false)
const formData = reactive({
  name: '',
  email: ''
})

// 计算属性
const isValid = computed(() => formData.name.length > 0)
</script>
```

### API 层

* 所有 API 调用封装在 `api/` 目录下
* 使用 Axios 实例，统一处理请求/响应拦截
* 请求参数和响应数据必须定义 TypeScript 类型
* 错误统一在拦截器中处理，返回类型化错误

```typescript
// api/user/index.ts
import { request } from '@/utils/request'
import type { UserInfo, UserListParams, UserListResult } from './model'

enum Api {
  UserList = '/system/user/page',
  UserDetail = '/system/user/get',
  UserCreate = '/system/user/create',
  UserUpdate = '/system/user/update',
  UserDelete = '/system/user/delete',
}

export function getUserList(params: UserListParams) {
  return request.get<UserListResult>(Api.UserList, { params })
}

export function getUserDetail(id: string) {
  return request.get<UserInfo>(`${Api.UserDetail}?id=${id}`)
}

export function createUser(data: Partial<UserInfo>) {
  return request.post<UserInfo>(Api.UserCreate, data)
}

export function updateUser(data: Partial<UserInfo>) {
  return request.put<UserInfo>(Api.UserUpdate, data)
}

export function deleteUser(id: string) {
  return request.delete(`${Api.UserDelete}?id=${id}`)
}
```

### 状态管理

* 使用 Pinia 进行状态管理
* Store 按业务模块划分，存放在 `store/modules/` 目录
* 复杂状态使用 Composition API 风格定义 Store
* 持久化状态使用 `pinia-plugin-persistedstate`

```typescript
// store/modules/user.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'
import { getUserInfo, login, logout } from '@/api/auth'
import type { UserInfo, LoginParams } from '@/api/auth/model'

export const useUserStore = defineStore(
  'user',
  () => {
    // State
    const token = ref<string>('')
    const userInfo = ref<UserInfo | null>(null)

    // Getters
    const isLoggedIn = computed(() => !!token.value)
    const permissions = computed(() => userInfo.value?.permissions ?? [])
    const roles = computed(() => userInfo.value?.roles ?? [])

    // Actions
    async function loginAction(params: LoginParams) {
      const result = await login(params)
      token.value = result.token
      await getUserInfoAction()
    }

    async function getUserInfoAction() {
      const info = await getUserInfo()
      userInfo.value = info
    }

    async function logoutAction() {
      await logout()
      token.value = ''
      userInfo.value = null
    }

    return {
      token,
      userInfo,
      isLoggedIn,
      permissions,
      roles,
      loginAction,
      getUserInfoAction,
      logoutAction,
    }
  },
  {
    persist: {
      key: 'user-store',
      paths: ['token'],
    },
  }
)
```

### 路由与权限

* 使用 Vue Router 4，路由配置按模块划分
* 动态路由由后端返回，前端根据权限动态注册
* 路由守卫处理登录验证和权限检查
* 页面组件使用 `definePage` 宏定义路由元信息

```typescript
// router/routes/modules/system.ts
import type { AppRouteRecordRaw } from '@/router/types'

const system: AppRouteRecordRaw = {
  path: '/system',
  name: 'System',
  component: LAYOUT,
  redirect: '/system/user',
  meta: {
    title: '系统管理',
    icon: 'ep:setting',
    order: 1,
  },
  children: [
    {
      path: 'user',
      name: 'SystemUser',
      component: () => import('@/views/system/user/index.vue'),
      meta: {
        title: '用户管理',
        icon: 'ep:user',
        permissions: ['system:user:list'],
      },
    },
    {
      path: 'role',
      name: 'SystemRole',
      component: () => import('@/views/system/role/index.vue'),
      meta: {
        title: '角色管理',
        icon: 'ep:user-filled',
        permissions: ['system:role:list'],
      },
    },
  ],
}

export default system
```

### 表单与验证

* 使用 Vee-Validate + Zod 进行表单验证
* 表单 Schema 与表单组件分离，便于复用
* 复杂表单使用分步或分组方式组织

```vue
<script setup lang="ts">
import { useForm } from 'vee-validate'
import { toTypedSchema } from '@vee-validate/zod'
import { z } from 'zod'

// 定义验证 Schema
const userSchema = toTypedSchema(
  z.object({
    name: z.string().min(2, '姓名至少2个字符').max(50, '姓名最多50个字符'),
    email: z.string().email('请输入有效的邮箱地址'),
    phone: z.string().regex(/^1[3-9]\d{9}$/, '请输入有效的手机号').optional(),
    deptId: z.number().min(1, '请选择部门'),
    status: z.enum(['0', '1']).default('0'),
  })
)

// 使用表单
const { handleSubmit, setValues, resetForm } = useForm({
  validationSchema: userSchema,
  initialValues: {
    name: '',
    email: '',
    status: '0',
  },
})

// 提交处理
const onSubmit = handleSubmit(async (values) => {
  try {
    loading.value = true
    await createUser(values)
    message.success('创建成功')
    emit('success')
  } catch (error) {
    console.error('创建用户失败:', error)
  } finally {
    loading.value = false
  }
})
</script>
```

### 代码风格

* 代码、注释或文档中不使用表情符号
* 组件保持在 500 行以内，复杂组件拆分为子组件
* 通用逻辑提取为 Composables
* 使用 ESLint + Prettier 统一代码风格
* 生产代码中不使用 `console.log`

## 文件结构

```
src/
├── api/                    # API 接口
│   ├── model/              # 类型定义
│   │   └── user.ts
│   └── system/             # 按模块划分
│       └── user/
│           ├── index.ts    # API 函数
│           └── model.ts    # 请求/响应类型
├── components/             # 通用组件
│   ├── Table/              # 表格组件
│   ├── Form/               # 表单组件
│   ├── Modal/              # 弹窗组件
│   └── UI/                 # 基础 UI 组件
├── composables/            # 组合式函数
│   ├── useLoading.ts       # 加载状态
│   ├── usePermission.ts    # 权限判断
│   ├── useTable.ts         # 表格封装
│   └── useForm.ts          # 表单封装
├── layouts/                # 布局组件
│   ├── default/            # 默认布局
│   └── blank/              # 空白布局
├── router/                 # 路由配置
│   ├── routes/             # 路由定义
│   │   └── modules/        # 按模块划分
│   ├── guard.ts            # 路由守卫
│   └── index.ts            # 路由实例
├── store/                  # 状态管理
│   └── modules/            # 按模块划分
│       ├── user.ts         # 用户状态
│       ├── permission.ts   # 权限状态
│       └── app.ts          # 应用状态
├── utils/                  # 工具函数
│   ├── request/            # Axios 封装
│   ├── dateUtil.ts         # 日期处理
│   └── treeUtil.ts         # 树形数据处理
├── views/                  # 页面视图
│   ├── dashboard/          # 仪表盘
│   ├── system/             # 系统管理
│   │   ├── user/           # 用户管理
│   │   ├── role/           # 角色管理
│   │   └── menu/           # 菜单管理
│   └── login/              # 登录页
├── types/                  # 全局类型定义
│   ├── global.d.ts
│   └── router.d.ts
├── enums/                  # 枚举定义
│   └── httpEnum.ts
├── hooks/                  # Vben 内置 hooks
└── settings/               # 配置文件
    ├── componentSetting.ts # 组件配置
    └── projectSetting.ts   # 项目配置
```

## 关键模式

### 列表页面模板

```vue
<script setup lang="ts">
import { ref, reactive } from 'vue'
import { useTable } from '@/composables/useTable'
import { getUserList, deleteUser } from '@/api/system/user'
import type { UserListParams, UserInfo } from '@/api/system/user/model'
import UserModal from './UserModal.vue'

// 搜索表单
const searchForm = reactive<Partial<UserListParams>>({
  username: '',
  status: undefined,
  deptId: undefined,
})

// 表格配置
const { register, reload, loading } = useTable({
  api: getUserList,
  columns: [
    { title: '用户名', dataIndex: 'username' },
    { title: '姓名', dataIndex: 'nickname' },
    { title: '部门', dataIndex: 'deptName' },
    { title: '状态', dataIndex: 'status', customRender: ({ text }) => text === '0' ? '正常' : '停用' },
    { title: '创建时间', dataIndex: 'createTime' },
    { title: '操作', dataIndex: 'action', width: 200 },
  ],
  searchForm,
  rowKey: 'id',
})

// 弹窗控制
const modalVisible = ref(false)
const modalRecord = ref<Partial<UserInfo>>()

function handleCreate() {
  modalRecord.value = undefined
  modalVisible.value = true
}

function handleEdit(record: UserInfo) {
  modalRecord.value = { ...record }
  modalVisible.value = true
}

async function handleDelete(record: UserInfo) {
  await deleteUser(record.id)
  reload()
}

function handleSuccess() {
  reload()
}
</script>

<template>
  <div class="page-container">
    <!-- 搜索区域 -->
    <a-card class="mb-4">
      <a-form :model="searchForm" layout="inline">
        <a-form-item label="用户名">
          <a-input v-model:value="searchForm.username" placeholder="请输入用户名" allow-clear />
        </a-form-item>
        <a-form-item label="状态">
          <a-select v-model:value="searchForm.status" placeholder="请选择状态" allow-clear>
            <a-select-option value="0">正常</a-select-option>
            <a-select-option value="1">停用</a-select-option>
          </a-select>
        </a-form-item>
        <a-form-item>
          <a-button type="primary" @click="reload">搜索</a-button>
          <a-button class="ml-2" @click="reset">重置</a-button>
        </a-form-item>
      </a-form>
    </a-card>

    <!-- 表格区域 -->
    <a-card>
      <template #extra>
        <a-button type="primary" @click="handleCreate">新增</a-button>
      </template>
      <a-table v-bind="register">
        <template #bodyCell="{ column, record }">
          <template v-if="column.dataIndex === 'action'">
            <a-button type="link" @click="handleEdit(record)">编辑</a-button>
            <a-popconfirm title="确定删除?" @confirm="handleDelete(record)">
              <a-button type="link" danger>删除</a-button>
            </a-popconfirm>
          </template>
        </template>
      </a-table>
    </a-card>

    <!-- 弹窗 -->
    <UserModal
      v-model:visible="modalVisible"
      :record="modalRecord"
      @success="handleSuccess"
    />
  </div>
</template>
```

### 表单弹窗模板

```vue
<script setup lang="ts">
import { ref, computed, watch } from 'vue'
import { useForm } from 'vee-validate'
import { toTypedSchema } from '@vee-validate/zod'
import { z } from 'zod'
import { createUser, updateUser } from '@/api/system/user'
import type { UserInfo } from '@/api/system/user/model'

const props = defineProps<{
  visible: boolean
  record?: Partial<UserInfo>
}>()

const emit = defineEmits<{
  (e: 'update:visible', value: boolean): void
  (e: 'success'): void
}>()

const isEdit = computed(() => !!props.record?.id)
const title = computed(() => (isEdit.value ? '编辑用户' : '新增用户'))

// 表单验证
const schema = toTypedSchema(
  z.object({
    username: z.string().min(3, '用户名至少3个字符').max(20, '用户名最多20个字符'),
    nickname: z.string().min(2, '姓名至少2个字符').max(50),
    email: z.string().email('请输入有效的邮箱'),
    phone: z.string().regex(/^1[3-9]\d{9}$/, '请输入有效的手机号').optional().or(z.literal('')),
    deptId: z.number({ required_error: '请选择部门' }),
    status: z.enum(['0', '1']).default('0'),
    roleIds: z.array(z.number()).optional(),
  })
)

const { handleSubmit, setValues, resetForm, errors } = useForm({
  validationSchema: schema,
})

// 监听 record 变化，填充表单
watch(
  () => props.record,
  (val) => {
    if (val) {
      setValues({
        username: val.username ?? '',
        nickname: val.nickname ?? '',
        email: val.email ?? '',
        phone: val.phone ?? '',
        deptId: val.deptId,
        status: val.status ?? '0',
        roleIds: val.roleIds ?? [],
      })
    } else {
      resetForm()
    }
  },
  { immediate: true }
)

const loading = ref(false)

const onSubmit = handleSubmit(async (values) => {
  try {
    loading.value = true
    if (isEdit.value) {
      await updateUser({ ...values, id: props.record!.id })
    } else {
      await createUser(values)
    }
    emit('success')
    emit('update:visible', false)
  } finally {
    loading.value = false
  }
})

function handleCancel() {
  emit('update:visible', false)
}
</script>

<template>
  <a-modal
    :open="visible"
    :title="title"
    :confirm-loading="loading"
    @ok="onSubmit"
    @cancel="handleCancel"
  >
    <a-form :label-col="{ span: 4 }" :wrapper-col="{ span: 20 }">
      <a-form-item label="用户名" :validate-status="errors.username ? 'error' : ''" :help="errors.username">
        <a-input v-model:value="values.username" placeholder="请输入用户名" :disabled="isEdit" />
      </a-form-item>
      <a-form-item label="姓名" :validate-status="errors.nickname ? 'error' : ''" :help="errors.nickname">
        <a-input v-model:value="values.nickname" placeholder="请输入姓名" />
      </a-form-item>
      <a-form-item label="邮箱" :validate-status="errors.email ? 'error' : ''" :help="errors.email">
        <a-input v-model:value="values.email" placeholder="请输入邮箱" />
      </a-form-item>
      <a-form-item label="手机号" :validate-status="errors.phone ? 'error' : ''" :help="errors.phone">
        <a-input v-model:value="values.phone" placeholder="请输入手机号" />
      </a-form-item>
      <a-form-item label="部门" :validate-status="errors.deptId ? 'error' : ''" :help="errors.deptId">
        <DeptTreeSelect v-model="values.deptId" />
      </a-form-item>
      <a-form-item label="状态">
        <a-radio-group v-model:value="values.status">
          <a-radio value="0">正常</a-radio>
          <a-radio value="1">停用</a-radio>
        </a-radio-group>
      </a-form-item>
    </a-form>
  </a-modal>
</template>
```

### Composables 封装

```typescript
// composables/useTable.ts
import { ref, reactive, unref, type Ref } from 'vue'
import type { TableProps } from 'ant-design-vue'

interface UseTableOptions<T> {
  api: (params: any) => Promise<{ items: T[]; total: number }>
  columns: any[]
  searchForm?: Record<string, any>
  rowKey?: string
  immediate?: boolean
}

export function useTable<T = any>(options: UseTableOptions<T>) {
  const { api, columns, searchForm = {}, rowKey = 'id', immediate = true } = options

  const loading = ref(false)
  const dataSource = ref<T[]>([]) as Ref<T[]>
  const pagination = reactive({
    current: 1,
    pageSize: 10,
    total: 0,
    showSizeChanger: true,
    showQuickJumper: true,
    showTotal: (total: number) => `共 ${total} 条`,
  })

  async function fetch() {
    try {
      loading.value = true
      const params = {
        ...unref(searchForm),
        pageNo: pagination.current,
        pageSize: pagination.pageSize,
      }
      const result = await api(params)
      dataSource.value = result.items
      pagination.total = result.total
    } finally {
      loading.value = false
    }
  }

  function reload() {
    pagination.current = 1
    fetch()
  }

  function reset() {
    Object.keys(searchForm).forEach((key) => {
      searchForm[key] = undefined
    })
    reload()
  }

  const tableProps: TableProps = {
    columns,
    dataSource,
    loading,
    pagination,
    rowKey,
  }

  if (immediate) {
    fetch()
  }

  return {
    register: tableProps,
    loading,
    reload,
    reset,
    fetch,
  }
}
```

## 环境变量

```bash
# 应用配置
VITE_APP_TITLE=管理后台
VITE_APP_BASE_API=/api

# 开发配置
VITE_DEV=true
VITE_PORT=3100

# 认证配置
VITE_TOKEN_PREFIX=Bearer
VITE_TOKEN_HEADER=Authorization

# 第三方服务
VITE_UPLOAD_URL=/api/file/upload
VITE_WS_URL=ws://localhost:8080/ws
```

## 测试策略

```bash
/vue-test             # Vue 组件 TDD 工作流
/vue-review           # Vue 代码审查
/build-fix            # 修复构建错误
```

### 测试命令

```bash
# 单元测试
pnpm test:unit

# 组件测试
pnpm test:component

# E2E 测试
pnpm test:e2e

# 测试覆盖率
pnpm test:coverage

# 类型检查
pnpm type:check

# 代码检查
pnpm lint

# 代码格式化
pnpm format
```

## ECC 工作流

```bash
# 规划
/plan "添加用户管理模块的导入导出功能"

# 开发
/vue-test                  # Vue 组件 TDD

# 审查
/vue-review                # Vue 最佳实践、性能优化
/security-scan             # 安全检查

# 提交前
pnpm lint
pnpm type:check
pnpm test:unit
pnpm build
```

## Git 工作流

* `feat:` 新功能，`fix:` 错误修复，`refactor:` 代码重构，`style:` 代码格式，`docs:` 文档更新，`test:` 测试相关
* 从 `main` 创建功能分支，需要 PR 审核
* CI: `pnpm lint`, `pnpm type:check`, `pnpm test`, `pnpm build`
* 部署: 构建 Docker 镜像或静态资源部署

## 常用命令

```bash
# 安装依赖
pnpm install

# 开发模式
pnpm dev

# 构建生产版本
pnpm build

# 预览生产版本
pnpm preview

# 代码检查
pnpm lint

# 类型检查
pnpm type:check

# 分析构建产物
pnpm build:analyze
```
