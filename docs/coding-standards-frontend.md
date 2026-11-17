# 前端编码规范（Vue 3 + TypeScript）

> 能被 lint / 编译器强制的内容交给配置（`eslint.config.js`、`tsconfig.json`），本文件只写**配置管不到但必须遵守**的部分。

## 1. 工程结构（apps/web）

```
apps/web/src
├── api/            # 只做 re-export：export * from '@repo/api-client/order'
├── assets/
├── components/     # 通用展示组件（无业务、无 store 依赖）
├── composables/    # useXxx.ts
├── layouts/
├── router/         # 按模块拆：routes/order.ts
├── stores/         # Pinia，仅放客户端状态
├── styles/
├── types/          # 应用层类型（接口类型从 @repo/shared-types 引）
├── utils/          # 纯函数，必须可单测
└── views/<domain>/ # 页面，按业务域分目录
```

依赖方向：`views → components / composables / stores → api-client / shared-types`，**禁止反向依赖**（`components` 不得 import `stores`）。

## 2. 命名

| 对象 | 规则 | 示例 |
|---|---|---|
| 组件文件 | PascalCase | `OrderTable.vue` |
| 组件 name | 与文件同名 | `defineOptions({ name: 'OrderTable' })` |
| composable | `useXxx` | `useTable.ts` |
| store | `useXxxStore` | `useOrderStore` |
| 布尔变量 | `is/has/can/show` 前缀 | `isLoading`、`showDialog` |
| 事件 | kebab-case | `@row-click`、`@update:model-value` |
| 常量 | UPPER_SNAKE | `PAGE_SIZE_OPTIONS` |

## 3. 组件写法模板

```vue
<script setup lang="ts">
import type { OrderVO } from '@repo/shared-types'
import { api } from '@repo/api-client'
import { ref } from 'vue'

interface OrderDetailProps {
  orderId: number
  readonly?: boolean
}

const props = withDefaults(defineProps<OrderDetailProps>(), { readonly: false })
const emit = defineEmits<{
  (e: 'change', order: OrderVO): void
  (e: 'close'): void
}>()

defineOptions({ name: 'OrderDetail' })

const order = ref<OrderVO | null>(null)
const isLoading = ref(false)

async function load() {
  isLoading.value = true
  try {
    const { data } = await api.order.get(props.orderId)
    order.value = data
    emit('change', data)
  } finally {
    // 必须在 finally 里复位加载态，禁止失败后按钮永久 loading
    isLoading.value = false
  }
}

void load()
</script>

<template>
  <el-skeleton v-if="isLoading" :rows="6" />
  <div v-else-if="order" data-testid="order-detail">…</div>
</template>
```

## 4. 请求层（packages/api-client）

```ts
// packages/api-client/src/http.ts
import axios, { type AxiosError } from 'axios'
import { ElMessage } from 'element-plus'
import type { Result } from '@repo/shared-types'

export const http = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL,
  timeout: 10_000,
})

// 请求：注入 token + 语言 + traceId
http.interceptors.request.use((cfg) => {
  const token = localStorage.getItem('access_token')
  if (token) cfg.headers.Authorization = `Bearer ${token}`
  cfg.headers['X-Trace-Id'] = crypto.randomUUID()
  return cfg
})

// 响应：统一拆 Result，code !== 0 一律抛错，只在这里提示
http.interceptors.response.use(
  (res) => {
    const body = res.data as Result<unknown>
    if (body.code !== 0) {
      ElMessage.error(body.msg ?? '请求失败')
      return Promise.reject(new BizError(body.code, body.msg))
    }
    return res
  },
  (err: AxiosError<Result<unknown>>) => {
    if (err.response?.status === 401) {
      // 交给路由守卫，不在这里跳转
      return Promise.reject(err)
    }
    ElMessage.error(err.response?.data?.msg ?? '网络异常，请稍后重试')
    return Promise.reject(err)
  },
)

export class BizError extends Error {
  constructor(readonly code: number, msg: string) {
    super(msg)
  }
}
```

约定：

- **错误提示只在拦截器或页面顶层出现一次**，业务代码里 `catch` 只做状态回滚，不再 toast。
- 类型：`Result<T>`、`PageResult<T>` 定义在 `@repo/shared-types`，与后端 DTO 一一对应（见 `docs/api/`）。
- 上传、下载、SSE 各自独立封装，不要塞进通用 `http`。

## 5. 状态管理

- **只放客户端状态**：登录用户、菜单元数据、UI 偏好、多步表单临时数据。
- **服务端数据不复制进 store**：用 `composables/useQuery.ts`（轻量封装）缓存 + 失效，避免两份真相。
- store 内 action 必须是唯一修改入口，组件禁止 `store.$patch` 直接改深层字段。

## 6. 路由与权限

- 路由 `meta` 必须有：`title`、`auth?: boolean`、`perm?: string`（权限点，与后端 `Permission` 枚举同名）。
- 按钮级权限用 `v-perm="'order:delete'"` 指令；**前端隐藏只是体验，后端必须再校验**。
- 路由守卫里取用户信息要有超时与失败兜底，禁止无限 `await`。

## 7. 性能与体验

- 大列表必须虚拟滚动或分页；表格单页上限 100。
- 路由级懒加载 + 首屏关键资源预加载；单个 chunk（gzip）> 300KB 需说明。
- 骨架屏/空态/错误态三件套必须实现，禁止"白屏 + console.error"。
- 所有表单提交按钮在请求期间 `loading` 且禁用，防重复提交。

## 8. 禁止项汇总

- `any`、`@ts-ignore`、`as unknown as X` 双重断言
- 组件内 `fetch` / 新建 axios 实例
- `v-html` 渲染用户输入
- `console.log`（生产构建会移除，但 PR 里出现即 nit → blocker，视情况）
- 手写 `new Date()` 格式化
- 在 setup 外使用 `onMounted` 等钩子