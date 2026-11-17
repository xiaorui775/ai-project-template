# ADR-0002 前端统一 Vue 3 + Vite + TS，不引入 React

- 状态：已采纳
- 日期：2026-09-05
- 决策人：@frontend-lead

## 背景

团队前端经验主要在 Vue 2 + Element UI。需要升级到现代栈，同时接入 AI 辅助开发。

## 可选方案

A. Vue 3 + Vite + TS + Pinia + Element Plus
B. React 18 + Next.js + TS + shadcn
C. 继续 Vue 2 + 渐进升级

## 决策

选 **A**。

## 理由

- 团队原有 Vue 经验可复用，迁移成本最低。
- 内部后台系统不需要 SSR / SEO，Vite SPA 足够，构建与调试更快。
- Element Plus 与已有设计规范一致，改造成本低。
- C 被否：Vue 2 已 EOL，且 AI 对 Vue 2 Options API 的生成质量下降。

## 后果

- 正面：上手快；生态与团队技能匹配。
- 负面：放弃 Next.js 的服务端渲染与文件路由（未来做 C 端站点时需重新评估）。
- 后续：所有新页面必须 `<script setup lang="ts">`，禁止再写 Options API。
