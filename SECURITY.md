# SECURITY.md —— 安全约束（人机共用红线）

> 本文件是**硬红线**。任何一条违反，PR 直接打回，不讨论。

## 1. 密钥与配置

- ❌ 禁止在代码、配置文件、测试、注释、文档、CI 脚本、prompt 中出现真实密钥。
- ✅ 所有敏感配置通过环境变量注入，`application.yml` 里只写 `${DB_PASSWORD}` 占位。
- ✅ 新增环境变量必须同步到项目的环境变量清单（如 `.env.example`），写清楚用途、是否必填、默认值。
- ✅ 前端只允许出现**公开**配置（`VITE_` 前缀会被打进产物，视为公开）。
- 检查命令：`git diff --cached | grep -Ei 'password|secret|token|BEGIN.*PRIVATE'`（pre-commit 已内置）。

## 2. 认证与授权（项目约定）

- 认证在 `gateway` 统一校验 JWT，业务服务从 `X-User-Id` / `X-Tenant-Id` 头取身份，**不重复解析 token**。
- 服务间内部调用：网关不暴露内部路径；内部接口统一 `/internal/**` 前缀，由网关拦截（返回 404）。
- **越权是最高频漏洞**：所有按 ID 查询/修改的接口，必须校验资源归属（`tenant_id` / `user_id`），禁止只靠前端隐藏按钮。
- 权限点枚举集中维护在 `services/common/.../constants/Permission.java`，禁止硬编码字符串 `"order:delete"`。

## 3. 输入与输出

- 后端：Controller 入参必须 `@Valid` + DTO 约束注解；文件上传校验大小、MIME、扩展名（白名单）和魔数。
- 后端：动态排序字段、动态表名、动态 SQL 片段一律白名单，禁止直接拼接用户输入。
- 前端：所有输出走 Vue 模板插值（自动转义）；**禁止 `v-html`**，确须富文本时用 `DOMPurify` 白名单净化。
- 前端：外部链接用 `rel="noopener noreferrer"`；不把 token 放进 URL query。

## 4. 数据与日志

- 日志脱敏字段（手机号、身份证、银行卡、token、密码）必须用 `@Sensitive` 注解处理。
- 日志禁止打印完整请求体（含 `Authorization` 头）；网关访问日志对其做 mask。
- 敏感数据落库需加密的字段：身份证、银行卡号（AES-GCM，密钥走 KMS / 环境变量）。
- 导出功能（Excel/PDF）需限制单次条数并记录审计日志。

## 5. 依赖与供应链

- 新增依赖前查 CVE（`./mvnw -q dependency:tree`、`pnpm audit`），高危直接拒绝，中危需 ADR 说明。
- 禁止引入 `*` 版本号（Maven）或 `latest`（npm）。
- CI 每天跑一次依赖扫描（`dependabot` / `dependency-check`），失败必须 3 天内处理。

## 6. 事务与并发（易被忽视的安全面）

- 跨服务写操作必须幂等：业务唯一键 + 状态机校验，禁止仅依赖 Seata 回滚。
- 金额、库存类字段禁止"读-改-写"覆盖，必须用乐观锁（`version`）或 `UPDATE ... SET x = x - n WHERE x >= n`。

## 7. AI 专属红线

- 禁止把生产数据、真实用户信息、真实密钥粘贴给 AI 工具或写进 prompt。
- AI 生成的安全相关代码（鉴权、加密、上传、SQL 拼接、反序列化）必须人工逐行评审。
- AI 不得为了"通过测试"而放宽校验（如删除 `@Valid`、关闭 CSRF、放开 CORS 为 `*`）。

## 8. 漏洞响应

发现疑似漏洞：私下联系安全负责人（见 `CODEOWNERS` 中 `SECURITY` 组），不要在公开 issue 里描述细节。