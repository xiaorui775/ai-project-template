# Pull Request

## 1. 改了什么

<!-- 一段话说明意图，不是文件的罗列 -->

## 2. 为什么

<!-- 关联需求 / issue / ADR，例如 REQ-001 或 ADR-0003 -->

Closes #

## 3. 怎么验证

<!-- 可复制的命令或步骤，评审者照着能复现 -->

```bash
pnpm lint && pnpm typecheck && pnpm test
./mvnw -q -pl services/<svc> -am verify
```

## 4. 风险与回滚

- 影响范围：
- 是否有 DB 变更（Flyway 版本号）：
- 是否有配置变更（Nacos dataId）：
- 回滚方式：revert commit / 关闭开关 / 回滚 Flyway：

## 自查清单

- [ ] 遵守 [AGENTS.md](../AGENTS.md) 第 4 节硬性约束
- [ ] 有测试覆盖新增/修改的逻辑，且测试不是"为了通过而写"
- [ ] 没有留 `TODO`、`console.log`、`System.out.println`、示例数据
- [ ] 新增依赖已在下方列出，并说明体积/许可证
- [ ] 数据库变更已写 Flyway 脚本，未含 `DROP` 等破坏性语句（或已单独说明）
- [ ] 涉及的接口文档（`docs/api/`）已同步
- [ ] 涉及架构决策的已补 ADR

## 新增依赖（无则填"无"）

| 包名 | 版本 | 用途 | 体积(gzip) | 许可证 |
|---|---|---|---|---|
| | | | | |