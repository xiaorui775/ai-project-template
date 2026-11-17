# CLAUDE.md

本项目的主约束文件是 [AGENTS.md](AGENTS.md)，请先完整阅读并严格遵守。

@AGENTS.md

## Claude Code 专属补充

以下内容只对 Claude Code 生效，其他工具可忽略。

### 子目录规则
- 进入 `apps/web/` 或 `packages/` 后，额外阅读 `docs/coding-standards-frontend.md`。
- 进入 `services/` 后，额外阅读 `docs/coding-standards-backend.md`。

### 工具使用约定
- 跑测试用 `pnpm test -- <pattern>` 或 `./mvnw -q -pl <module> -am test -Dtest=<Class>`，**不要跑全量**（耗时 > 5 分钟）。
- 修改 `pom.xml` 后必须执行 `./mvnw -q -pl <module> -am dependency:resolve` 确认依赖能解析。
- 需要联网查文档时，优先查官方站点（Spring / Vue / Element Plus），不要把博客里的过时代码贴进来。

### 回答格式
- 代码块前用一句话说明"这个改哪个文件、为什么"。
- 多个方案时给对比表（改动量 / 风险 / 回滚难度），**不要直接替我做选择**（除明确的小改动外）。
- 结束时按 AGENTS.md 第 5 节给出三段式总结。
