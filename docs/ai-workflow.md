# AI 协作工作流（人机协作方式）

> 作用：定义**怎么和 AI 一起干活**，而不是 AI 该写什么代码（那是 `AGENTS.md`）。

## 1. 任务颗粒度

| 类型 | 建议交给 AI 的粒度 | 人工负责 |
|---|---|---|
| 新增 CRUD 接口 | 一个接口（DTO + Service 方法 + Controller + 测试） | 评审事务边界、权限 |
| 新页面 | 一个页面 + 一个 composable | 交互设计、视觉验收 |
| Bug 修复 | 必须先给 AI 复现步骤 + 期望值 | 判定根因是否被修对 |
| 重构 | 一次一个函数/类 | 决定要不要重构 |
| 架构变更 | ❌ 不交给 AI 自主完成 | 人来决策 + 写 ADR |

**不要**把"实现整个需求"一句话丢给 AI；拆成上面这种可验收的小块。

## 2. 标准对话模板

```
【背景】<相关文件路径 + 一句话现状>
【目标】<可验收的一句话>
【约束】<必须遵守的点：不能改 X，必须兼容 Y>
【验收】<跑什么命令 / 看什么结果>
【不要做】<明确排除项>
```

示例：

```
【背景】services/order 的 OrderService.create 目前直接调 InventoryClient.deduct。
【目标】余额扣减失败时订单不落库，且返回明确错误码 30021。
【约束】不要引入新依赖；不要改 Controller 层。
【验收】新增单测 OrderServiceImplTest#should_rollback_when_deduct_failed 通过；./mvnw -pl services/order -am test 全绿。
【不要做】不要顺手重构 OrderConvert。
```

## 3. 让 AI 先读、再写

- 在 prompt 里显式指定"先读哪些文件"：`先读 services/order/.../OrderService.java 和 docs/coding-standards-backend.md`。
- 修改公共层（`common`）前必须要求 AI 列出**所有调用方**。

## 4. 验收标准（AI 交付必须满足）

1. 编译通过、lint 通过、测试通过。
2. 新增逻辑有测试，且**至少一个失败路径**。
3. 改动的每个 import 都真实存在（防幻觉）。
4. 变更摘要三段式：改了什么 / 怎么验证 / 剩余风险。
5. 若触及规范空白，已在 `docs/` 补一条。

## 5. 何时必须停下来问人

- 需求有歧义（两种解释都合理）
- 需要新增依赖、新中间件、新表
- 需要改动 `AGENTS.md` 里的"已定技术选型"
- 需要放宽安全约束（关校验、放开 CORS、禁用 CSRF）
- 涉及删除数据、改数据结构、改权限模型

## 6. 定期维护约束文档

- 每次事故/返工后：补 `docs/known-pitfalls.md` 一条。
- 每条规则若 3 个月内从未触发过，考虑删除（规则越多，AI 遵守率越低）。
- 根 `AGENTS.md` 超过 500 行就该拆（下沉到 scoped rules 或 docs）。