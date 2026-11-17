# Copilot 指令

> 主约束见 [AGENTS.md](../AGENTS.md)。本文件**只写 Copilot 补全场景的偏好**，不重复规则。

项目为 Vue 3 + TypeScript 前端 / Spring Boot 3 + Spring Cloud Alibaba 后端，pnpm + Maven 多模块同仓。

## 补全时优先遵循

- **前端**：`<script setup lang="ts">` + 组合式 API；组件具名导出；类型从 `@repo/shared-types` 导入，不要现场声明后端 VO。
- **前端**：请求一律 `api.xxx`（来自 `@repo/api-client`），不要补出 `axios.get` / `fetch`。
- **后端**：Controller 只做校验与转发，返回 `Result<T>`；业务在 Service；跨服务调用走 Feign 接口。
- **后端**：DTO / VO 用 record 或 Lombok `@Data`，字段带 `@Schema(description)`。

## 不要补出来的东西

- 示例数据、`// TODO 实现`、`throw new UnsupportedOperationException()`
- `System.out.println`、`e.printStackTrace()`、`console.log`
- `any`、`@ts-ignore`、`SELECT *`
- 硬编码的密钥、地址、账号

## 命名

| 场景 | 规则 |
|---|---|
| Java 类 | `XxxController` / `XxxService` / `XxxServiceImpl` / `XxxMapper` / `XxxVO` / `XxxDTO` |
| Java 方法 | 动词开头：`createOrder` / `queryById` / `pageInvoices` |
| TS 组件 | PascalCase 文件 + `defineOptions({ name })` 同名 |
| TS 组合式函数 | `useXxx` |

生成代码后如需细节约束，查阅 [docs/coding-standards-frontend.md](../docs/coding-standards-frontend.md) 与 [docs/coding-standards-backend.md](../docs/coding-standards-backend.md)。