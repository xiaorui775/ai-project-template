---
applyTo: "services/**/*.java,services/**/*.xml,services/**/*.yml"
---

# 后端目录级指令

进入本路径后，除 [AGENTS.md](../../AGENTS.md) 外，还必须遵守 [docs/coding-standards-backend.md](../../docs/coding-standards-backend.md) 与 [docs/architecture.md](../../docs/architecture.md)。

## 检查点

- [ ] Controller 无业务逻辑，返回 `Result<T>`
- [ ] 入参 `@Valid`，接口有 `@Operation` 说明
- [ ] 异常抛 `BizException`，不吞异常、不 `printStackTrace`
- [ ] 查询显式列名，无 `SELECT *`
- [ ] 列表接口分页，禁止一次性返回全量
- [ ] 跨服务调用走 Feign，已配置 fallback 与超时
- [ ] 跨服务写事务已评估是否需要 `@GlobalTransactional`
- [ ] DDL 走 Flyway，未手改数据库
- [ ] 日志不含手机号、身份证、token 原文

## 分层职责

| 层 | 允许做 | 禁止做 |
|---|---|---|
| Controller | 参数校验、调用 Service、组装 `Result` | 写业务分支、直接调 Mapper |
| Service | 业务规则、事务边界、编排 | 直接拼 SQL 字符串 |
| Mapper | 数据访问 | 业务判断 |
| Feign Client | 远程调用封装 | 写业务逻辑 |

## 事务边界

- 单库写：Service 方法上 `@Transactional(rollbackFor = Exception.class)`
- 跨服务写：外层 `@GlobalTransactional`，内层本地事务照常
- **不要在 Controller 上开事务**