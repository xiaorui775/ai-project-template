# ai-project-template

> **本仓库只提供约束文档**，不含业务代码，也不含 `package.json` / `pom.xml` 等构建配置。

面向 **Vue 3 + TypeScript 前端 / Spring Boot 3 + Spring Cloud Alibaba 后端** 的 AI Coding 约束文档集。

所有文件名都是**生产环境可直接使用的正式名字**：复制到新仓库根目录即可生效，不需要改名、不需要删标记。

## 两个入口

| 你是 | 读这份 |
|---|---|
| **新项目开工，想直接用** | 本文件（30 秒上手 + 文件清单） |
| **想理解、维护这套约束**（哪些必选、每个文件写什么、怎么改） | [模板使用说明.md](模板使用说明.md) |

**哪些是必选**：见 [模板使用说明.md §3 必选文档](模板使用说明.md#31-必选文档清单23-个)，那里逐个说明了「为什么必选」「缺了会怎样」「能不能裁剪」。

## 30 秒上手

1. 把本目录内容复制到新仓库根目录（**含** `.github/`、`.cursor/`、`.gitignore` 与隐藏文件）。
2. 改 `AGENTS.md` 第 1/2/3 节里的**技术栈、版本、命令**——这是唯一必须改的地方。
3. 删掉不适用的部分（例如不用 Seata 就删掉 `backend-microservice.mdc` 的事务表）。
4. 把 `.github/CODEOWNERS` 里的 `@company/*` 换成真实 team。
5. 按 [模板使用说明.md §3](模板使用说明.md#3-文件总表意义--位置--层级) 补齐「推荐」「按需」项。
6. 第一个真实需求到来时，复制 `docs/PRD.md` 成 `docs/requirements/REQ-001-xxx.md` 再填。

## 文件清单（37 个）

| 分组 | 文件 | 作用 |
|---|---|---|
| **入口** | `AGENTS.md` | 唯一信源：技术栈、命令、硬性约束、AI 行为约定、文档地图 |
| | `CLAUDE.md` | Claude Code 入口，`@AGENTS.md` 引用 + 工具专属补充 |
| | `.cursorrules` | 旧版 Cursor 入口，只做转发 |
| **红线** | `SECURITY.md` | 安全红线：密钥、鉴权、越权、SQL、XSS、脱敏、AI 红线 |
| | `CONTRIBUTING.md` | 分支、Commit、PR、评审优先级、发布顺序 |
| **规范** | `docs/coding-standards-frontend.md` | 前端目录、命名、组件模板、请求层、状态、权限、禁止项 |
| | `docs/coding-standards-backend.md` | 后端模块划分、包结构、分层模板、必守细则、常见错误 |
| | `docs/testing.md` | 测试分层比例、命名、前后端示例、测试数据、禁止项 |
| | `docs/ai-workflow.md` | 人机协作方式：任务颗粒度、prompt 模板、何时停下来问人 |
| **需求与架构** | `docs/PRD.md` | 需求模板 + 含 Given/When/Then 验收标准的完整示例 |
| | `docs/architecture.md` | 服务边界、数据属主、调用链、一致性策略 |
| **决策与记录** | `docs/adr/README.md` / `template.md` / `0001~0004` | ADR 索引、模板、4 条示例决策 |
| | `docs/known-pitfalls.md` | 踩坑清单（现象 → 根因 → 正确做法），只追加 |
| **运维** | `docs/runbooks/service-down.md` / `config-change.md` | 服务不可用处置、配置变更流程 |
| **路径规则** | `.cursor/rules/frontend.mdc` | Cursor 前端规则（按 glob 生效） |
| | `.cursor/rules/backend.mdc` | Cursor 后端规则 |
| | `.cursor/rules/backend-microservice.mdc` | Feign / 事务边界 / 限流 / 网关 |
| | `.cursor/rules/testing.mdc` | 测试编写规则 |
| | `.github/copilot-instructions.md` | Copilot 补全偏好 |
| | `.github/instructions/frontend.instructions.md` | Copilot 前端目录级规则（`applyTo`） |
| | `.github/instructions/backend.instructions.md` | Copilot 后端目录级规则（`applyTo`） |
| **机器兜底** | `.github/workflows/ci.yml` | CI 四道闸门 + 汇总 job |
| | `.github/CODEOWNERS` | 高危目录强制双签 |
| | `.github/pull_request_template.md` | PR 四段结构（变更 / 验证 / 风险 / 回滚） |
| | `.gitignore` | 用文件系统表达的约束：什么永远不该进仓库 |
| **按场景** | `docs/glossary.md` | 领域术语表，减少 AI 误读 |
| | `docs/api/invoice.md` | 单接口契约示例（字段 / 错误码 / 幂等） |
| | `docs/mcp.md` | MCP 工具白名单与接入审批流程 |
| | `CHANGELOG.md` | 变更记录写法（含 BREAKING CHANGE 迁移说明） |
| **元文档** | `README.md` / `模板使用说明.md` | 本文件、以及每个约束文档的意义 / 位置 / 编写规范 / 维护流程 |

## 设计原则

1. **单一信源**：规则只写一份，其他工具文件做引用，避免改一处剩九处过期。
2. **可执行优先**：能写成 lint / CI 规则的就不写文字；文字只保留判断类约束（"什么时候该停下来问人"）。
3. **短而分层**：根文件 150–300 行，细则下沉到 `.cursor/rules` 与 `docs/`。
4. **诚实写负面后果**：ADR 里必须写"选它的代价"，这样 AI 才不会反复提议被否决的方案。

## 明确没有放进来的

- 业务代码本身（只给模板与片段，避免 AI 照抄示例）
- 构建工具配置（`package.json`、`pom.xml`、`.nvmrc`、`.env.example` 等）：构建配置由真实项目自己定义，文档里提到的命令与路径请按实际项目改写（`.gitignore` 属于纯约束、予以保留）
- 生产部署配置（`deploy/` 只提到，不给内容）
- 许可证与合规文档（按公司要求另加）
