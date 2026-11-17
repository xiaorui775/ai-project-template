---
applyTo: "apps/web/**/*.{ts,vue},packages/**/*.ts"
---

# 前端目录级指令

进入本路径后，除 [AGENTS.md](../../AGENTS.md) 外，还必须遵守 [docs/coding-standards-frontend.md](../../docs/coding-standards-frontend.md)。

## 检查点

- [ ] 类型来自 `@repo/shared-types`，没有手写后端字段
- [ ] 请求走 `@repo/api-client`，组件内无 `axios` / `fetch`
- [ ] 组件具名导出 + `defineOptions({ name })`
- [ ] 无 `any` / `@ts-ignore` / `console.log`
- [ ] 页面具备加载态、空态、错误态
- [ ] 雪花 ID 当 `string` 处理，金额当 `string` 处理

## 目录职责

| 目录 | 放什么 |
|---|---|
| `src/views/` | 页面，按业务域分目录 |
| `src/components/` | 本应用通用组件 |
| `src/composables/` | 组合式函数，`useXxx` 命名 |
| `src/stores/` | Pinia，只放客户端状态 |
| `src/api/` | 只做 re-export |
| `src/constants/` | 常量与码表 |
| `src/types/` | 应用层类型（不写契约类型） |

## 反例 → 正例

```ts
// ❌ 组件里直连后端
const res = await axios.get(`/api/v1/order/${id}`)
order.value = res.data.data

// ✅
const { data } = await api.order.get(id)
order.value = data
```