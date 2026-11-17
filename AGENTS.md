# AGENTS.md

## 1. 项目概览

企业级前后端分离项目，**Maven 多模块（Spring Cloud Alibaba）+ pnpm Monorepo（Vue 3）** 同仓库管理。

```
.
├── apps/web/              # Vue 3 + Vite 前端应用
├── packages/              # 前端共享包（ui / api-client / shared-types / utils）
├── services/              # Spring Boot 3 服务（每个服务一个 Maven 模块）
│   ├── gateway/           # Spring Cloud Gateway 网关
│   ├── auth/              # 认证授权服务
│   ├── order/             # 业务服务示例
│   └── common/            # 公共模块（结果封装、异常、工具、starter）
└── docs/                  # 需求、架构、规范、决策记录
```

技术栈（**已定，勿擅自更换**）：

| 层 | 选型 |
|---|---|
| 前端 | Vue 3.5 `<script setup>` + Vite 6 + TypeScript 5.7 + Pinia + Vue Router 4 + Element Plus |
| 前端请求 | axios（统一封装于 `packages/api-client`） |
| 后端 | JDK 21 + Spring Boot 3.3 + Spring Cloud 2023.0 + Spring Cloud Alibaba 2023.0 |
| 注册/配置 | Nacos |
| 网关 | Spring Cloud Gateway |
| 调用 | OpenFeign + LoadBalancer |
| 熔断限流 | Sentinel |
| 分布式事务 | Seata（AT 模式，仅跨服务写事务使用） |
| ORM | MyBatis-Plus |
| 数据库 | MySQL 8.0 / Redis 7 |
| 构建 | Maven 3.9（mvnw）+ pnpm 9 |

详见 [docs/architecture.md](docs/architecture.md)。

## 2. 运行环境

- JDK **21**，Maven 用 `./mvnw`，**不要直接调用系统 mvn**。
- Node **22.11.x**，pnpm **9.x**。
- **包管理器只用 pnpm**：禁止 npm / yarn，禁止提交后手工改 `pnpm-lock.yaml`。
- 后端依赖版本统一由根 `pom.xml` 的 `dependencyManagement` 管理，**子模块禁止写 `<version>`**。

## 3. 常用命令

```bash
# ---------- 前端 ----------
pnpm install                 # 根目录安装全部 workspace 依赖
pnpm dev                     # 启动 web（Vite，默认 5173）
pnpm --filter web build      # 构建
pnpm lint                    # ESLint
pnpm typecheck               # vue-tsc --noEmit
pnpm test                    # vitest
pnpm test:e2e                # playwright

# ---------- 后端 ----------
./mvnw -q -pl services/order -am spring-boot:run    # 启动单个服务
./mvnw -q -DskipTests package                        # 全量打包
./mvnw -q -pl services/order -am test                # 单模块测试
./mvnw -q spotless:apply                             # 格式化（Google Java Format）
./mvnw -q verify                                     # 含 checkstyle

# ---------- 联调 ----------
docker compose -f deploy/docker-compose.dev.yml up -d   # Nacos / MySQL / Redis / Sentinel
```

提交前**必须全绿**：`pnpm lint && pnpm typecheck && pnpm test` + `./mvnw -q verify`。

## 4. 硬性约束（违反即视为错误，评审直接打回）

### 4.1 通用
- 一次改动**只做一件事**；PR diff 超过 400 行必须先提方案，不要直接写。
- 不要重构与需求无关的代码；不要全文件格式化，只格式化改动行。
- 不确定的 API / 字段名：**先读代码或类型定义，再动手**；查不到就停下来提问，**禁止臆造方法名、字段名**。
- 禁止输出 `// ...` 省略号占位、`TODO` 空实现、假数据 mock 混在生产代码里。

### 4.2 前端
- 禁止 `any`、`@ts-ignore`、`@ts-expect-error`（确需时先询问；用 `unknown` + 类型守卫替代）。
- 禁止在组件里直接 `fetch` / 直接 `axios.create`，统一走 `packages/api-client`。
- 禁止 `default export` 组件；组件用具名导出 + `defineOptions({ name: 'XxxYyy' })`。
- 禁止 `v-html` 渲染用户输入（XSS）；确需时先过 `DOMPurify` 并注释说明。
- 禁止在 `setup` 之外改 Pinia store；组件内禁止直接 `store.$state = ...`。
- 新增依赖需在 PR 描述里说明体积（gzip 后 KB）与许可证。

### 4.3 后端
- 禁止 `System.out.println`、`e.printStackTrace()`；异常统一走 `GlobalExceptionHandler`。
- 禁止在 `Controller` 里写业务逻辑；Controller 只做参数校验 + 调用 Service + 返回 `Result<T>`。
- **跨服务调用禁止直连数据库**：只能通过 Feign 接口 + 服务名（`http://order-service`）。
- **跨服务写事务**：涉及 ≥2 个服务写库时，必须用 `@GlobalTransactional`（Seata），并在设计文档中说明回滚点。
- 禁止 `SELECT *`（MyBatis XML 里显式列名），禁止 N+1 查询进循环。
- 禁止手改数据库表：一切 DDL 走 Flyway（`services/<svc>/src/main/resources/db/migration/V*__xxx.sql`）。
- Feign 客户端必须配置 fallback + 超时；不加 fallback 的 Feign 调用视为缺陷。
- 所有对外接口必须写 `@Operation`（springdoc）注释，并用 `@Valid` 校验入参。

### 4.4 安全
完整规则见 [SECURITY.md](SECURITY.md)，红线如下：
- 密钥/密码/token **禁止硬编码**，一律读 `application.yml` + 环境变量，新增变量同步到环境变量清单文档。
- 日志禁止打印手机号、身份证、token 原文（用 `@Sensitive` 脱敏）。
- 禁止字符串拼接 SQL；MyBatis 只用 `#{}`，禁止 `${}`（排序字段需白名单）。
- 新增依赖必须查 CVE，高危直接拒绝。

## 5. 工作方式（AI 行为约定）

1. **动手前**：用 3–5 行复述你的理解和改动计划（涉及哪些文件、为什么），等确认后再写。
2. **改动范围**：一次最多改 3 个文件；超出先说明理由。
3. **需求不清**：停下来提问，列出 2–3 个具体选项让我选，不要自选一个默默实现。
4. **修 bug**：先写能复现的失败测试，再修。
5. **完成时**必须在回复里给出：
   - 改了什么（文件 + 一句话）
   - 怎么验证（可执行的命令）
   - 还有什么没做 / 已知风险
6. 新增领域术语先补 [docs/glossary.md](docs/glossary.md)，新增技术决策先写 ADR（[docs/adr/](docs/adr/)）。

## 6. 文档地图（什么时候读哪份）

| 场景 | 读这份 |
|---|---|
| 不知道怎么和 AI 配合干活 | [docs/ai-workflow.md](docs/ai-workflow.md) |
| 不知道需求边界 / 验收标准 | [docs/PRD.md](docs/PRD.md) |
| 不清楚服务拆分、调用链 | [docs/architecture.md](docs/architecture.md) |
| 写 Java 代码 | [docs/coding-standards-backend.md](docs/coding-standards-backend.md) |
| 写 TS / Vue 代码 | [docs/coding-standards-frontend.md](docs/coding-standards-frontend.md) |
| 写测试 | [docs/testing.md](docs/testing.md) |
| 碰鉴权、上传、SQL、密钥 | [SECURITY.md](SECURITY.md) |
| 线上问题排查 | [docs/runbooks/](docs/runbooks/) |
| 这个坑以前踩过吗 | [docs/known-pitfalls.md](docs/known-pitfalls.md) |
| 为什么这么选 | [docs/adr/](docs/adr/) |
| 名词看不懂 | [docs/glossary.md](docs/glossary.md) |
| 提交 / 分支 / 评审流程 | [CONTRIBUTING.md](CONTRIBUTING.md) |

## 7. 不要做

- 不要擅自升级 Spring Boot / Spring Cloud / Vue 的版本。
- 不要在 `common` 模块里塞业务代码（只放与业务无关的基础设施）。
- 不要为了"让测试通过"而删除或 skip 测试；不要改测试断言来迁就错误实现。
- 不要自动 push、merge、改 CI 配置、改 `deploy/` 下的生产配置。