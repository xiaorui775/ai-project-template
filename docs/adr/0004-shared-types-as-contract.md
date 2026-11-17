# ADR-0004 前端类型由 packages/shared-types 作为契约

- 状态：已采纳
- 日期：2026-10-08
- 决策人：@frontend-lead

## 背景

前后端类型不同步是最高频的问题：后端加了字段前端不知道，后端把 `Long` 改 `String` 前端类型没变，运行时才炸。

## 可选方案

A. 手写两份（前端 types 单独维护）
B. 从后端生成 OpenAPI 再 codegen 到 TS
C. 手写一份 `packages/shared-types`，由 CI 做契约校验

## 决策

选 **C**，并保留升级到 B 的路径。

## 理由

- 当前接口量不大（约 40 个），手写成本低且可控（可以写更精确的联合类型）。
- codegen 出的类型通常 `any` 泛滥、命名丑，反而降低 AI 生成质量。
- CI 中解析后端 springdoc 的 `/v3/api-docs`，对比 TS 里的字段，缺失即失败，能挡住 80% 的漂移。

## 后果

- 正面：AI 生成前有真类型可读；编译期就能发现字段不一致。
- 负面：改接口要改三处（后端 DTO / docs/api / shared-types），流程略重。
- 后续：接口量 > 100 时重新评估 codegen（届时写新 ADR 取代本条）。
