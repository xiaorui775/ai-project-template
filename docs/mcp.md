# MCP 工具接入约定

> 本文件是 `MCP server` 的**白名单与审批流程**。模板仓库不提供 `.mcp.json`，请在真实项目里自行创建。

## 为什么单独一份文档

MCP 配置文件是 JSON 格式，**不能写注释**（写注释会导致客户端解析失败）。所以"为什么允许这个 server、谁批的、能读什么"这些说明必须放在 md 里，配置里只保留纯结构。

## 白名单

| server | 用途 | 作用域 | 风险 | 审批人 |
|---|---|---|---|---|
| filesystem | 读写仓库内文件 | 仅仓库路径 | 低 | 无需单独审批 |
| dbhub-dev | 只读查询**开发库** | 只读账号，禁用 DDL | 中 | 后端负责人 |
| context7 | 查询官方文档 | 公网只读 | 低 | 无需单独审批 |

新增一行 = 新增一个 AI 能碰到的外部系统，必须走下面流程。

## 接入流程

新增 MCP server 等于新增攻击面。必须依次完成：

1. **提 PR 说明用途**：它能读什么、能写什么、失败了会怎样。
2. **安全负责人 review**（见 `CODEOWNERS` 中的 security 组）。
3. **更新本文件的白名单表**，写清作用域与风险等级。
4. **最后才改** `.mcp.json` 或各客户端的 MCP 配置。

## 硬性禁止

- ❌ 任何含真实密钥的 DSN / token 直接写进配置；配置里只能出现环境变量占位符（如 `${DEV_DB_DSN}`）。
- ❌ 连接生产库的 server，哪怕只是只读——需要时临时开、用完立刻关，且有人值守。
- ❌ 允许执行任意 shell 的 server。
- ❌ 未经审批直接在本机配置里加 server 后提交。

## 示例配置（纯结构，占位符形式）

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "${WORKSPACE_ROOT}"]
    },
    "dbhub-dev": {
      "command": "npx",
      "args": ["-y", "@bytebase/dbhub", "--dsn", "${DEV_DB_DSN}"]
    }
  }
}
```

## 我认为 AI 不应该做的事

MCP 扩大了 AI 的能力边界，因此额外约定：

- AI 通过 MCP 读到的数据**不得**粘回 prompt 或提交进仓库（尤其是脱敏前的业务数据）。
- AI 不得通过 MCP 修改数据库结构，DDL 只能走 Flyway。
- 每次新增 server 后，在 `docs/known-pitfalls.md` 里记一条"踩过的坑"是推荐做法。
