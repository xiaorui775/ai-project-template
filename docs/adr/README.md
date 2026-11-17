# ADR 索引

架构决策记录（Architecture Decision Record）。**一个决策一个文件，一旦写入不再修改内容**；要推翻就新写一条并标注"取代 ADR-XXXX"。

AI 规则：提出被否决过的方案前，先查这里。

| 编号 | 标题 | 状态 | 日期 |
|---|---|---|---|
| [0001](0001-use-maven-multimodule.md) | 后端使用 Maven 多模块而非独立仓库 | 已采纳 | 2026-09-02 |
| [0002](0002-vue3-vite-ts.md) | 前端统一 Vue 3 + Vite + TS，不引入 React | 已采纳 | 2026-09-05 |
| [0003](0003-sync-vs-async-consistency.md) | 跨服务一致性优先最终一致，Seata 为兜底 | 已采纳 | 2026-09-18 |
| [0004](0004-shared-types-as-contract.md) | 前端类型由 `packages/shared-types` 作为契约 | 已采纳 | 2026-10-08 |
| [template.md](template.md) | 新建 ADR 时复制本模板 | — | — |