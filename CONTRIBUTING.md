# 贡献指南（人机共用）

> AI 也必须遵守本文件。冲突时以 [AGENTS.md](AGENTS.md) 为准。

## 1. 分支模型

| 分支 | 用途 | 保护 |
|---|---|---|
| `main` | 生产，随时可发布 | 受保护，禁止直接 push / force push |
| `develop` | 集成 | 受保护，需 CI 全绿 |
| `feat/<ticket>-<slug>` | 功能 | — |
| `fix/<ticket>-<slug>` | 修复 | — |
| `hotfix/<ticket>` | 线上紧急修复，从 `main` 拉出 | — |

AI 生成的分支名必须带 ticket 号，例如 `feat/PROJ-123-invoice-export`。

## 2. 提交信息（Conventional Commits）

```
<type>(<scope>): <简述，中文，不超过 50 字>

<可选正文：为什么改，不写"改了什么"——diff 已经说明了>
<可选：影响面 / 回滚方式>

Refs: PROJ-123
Co-authored-by: <AI 工具名> <noreply@example.com>
```

- `type`：`feat` `fix` `docs` `refactor` `test` `chore` `perf` `build` `ci`
- `scope`：`web` `api-client` `gateway` `auth` `order` `common` `infra` `docs`
- **AI 参与生成的提交必须带 `Co-authored-by` 行**，便于审计。

## 3. 提交前检查（本地必须全绿）

```bash
pnpm lint && pnpm typecheck && pnpm test
./mvnw -q spotless:apply verify
```

`pre-commit` 钩子会拦截：ESLint 错误、格式化未执行、提交信息格式错误、误提交 `.env`。

## 4. PR 要求

PR 描述使用 `.github/pull_request_template.md`，必须包含四段：

1. **变更摘要**：一句话说清做了什么。
2. **验证方式**：可复现的步骤或命令（最好带截图 / 测试名）。
3. **风险与影响面**：涉及哪些服务、是否有 DB 迁移、是否需要配置变更、是否要灰度。
4. **回滚方案**：具体到命令或开关。

硬性规则：

- 一个 PR 只做一件事；diff > 400 行需先拆或先出方案。
- **DB 迁移单独一个 PR**，且必须向后兼容（先加列后删列，分两次发布）。
- 变更 `pom.xml` / `pnpm-lock.yaml` 时，PR 里说明新增依赖的用途与体积。
- 涉及接口改动 → 同步更新 `docs/api/` 与前端 `packages/shared-types`。

## 5. 评审

- 至少 1 人 approve；`services/gateway`、`services/auth`、`deploy/`、`SECURITY.md`、根 `pom.xml` 需 CODEOWNERS 中对应 owner approve。
- 评审关注点（按优先级）：
  1. 是否有安全红线问题（密钥、SQL 注入、越权、XSS）
  2. 事务与一致性边界是否正确（是否该用 Seata、幂等是否成立）
  3. 跨服务调用是否有 fallback、超时、重试策略
  4. 测试是否覆盖失败路径
  5. 命名与分层是否符合规范
- 阻塞项用 `[blocker]`，建议项用 `[nit]`，AI 修改时**只处理 blocker，nit 需确认后再动**。

## 6. 发布

- 版本号：`MAJOR.MINOR.PATCH`，前端与后端各自独立打 tag：`web-v1.4.0`、`svc-order-v2.1.3`。
- 破坏性变更必须在 `CHANGELOG.md` 标注 `BREAKING CHANGE` 并给出迁移步骤。
- 发布顺序：DB 迁移 → 后端（灰度 Nacos 权重 10% → 50% → 100%）→ 前端。

## 7. AI 参与的额外约定

- AI 不得自动 `push`、`merge`、修改 CI 配置、修改 `deploy/` 生产配置。
- AI 生成的新依赖、新中间件、新表结构，必须先写 ADR 再动代码。
- AI 完成一个任务后，若发现规范缺失（比如某个场景没约定），应当在 `docs/` 里补一条，而不是只在对话里说。