# Runbook：配置变更（Nacos）

## 原则

- 配置变更是**最高频的事故来源**。所有变更必须留痕，且可一步回滚。
- 敏感配置（密码、密钥）**不进 Nacos**，走环境变量 / KMS。

## 变更流程

1. 在 `docs/config/<服务>-<环境>.md` 记录：改了什么、为什么、影响面、回滚值。
2. Nacos 控制台修改 → **先改 dev**，观察 10 分钟。
3. test 环境验证后改 prod；prod 变更尽量在低峰期（避开 00:00–02:00 批量任务）。
4. 观察指标：错误率、QPS、RT、GC。异常立即回滚。

## 回滚

Nacos 配置页 → 历史版本 → 选择上一个版本 → 回滚 → 确认实例收到推送（`/actuator/env` 或日志 `Refresh keys changed`）。

## 使用 `@RefreshScope` 的坑

- 带 `@RefreshScope` 的 Bean 在刷新时会**重建**，若持有连接池/线程池会导致泄漏。
- 需要动态的配置优先用 `@ConfigurationProperties`（Spring Cloud 会整体 rebind），或直接从 `Environment` 读取。
- 开关类配置建议放 Redis（`config:switch:xxx`），避免全量推送。

## 变更记录模板

| 时间 | 环境 | 配置项 | 原值 | 新值 | 原因 | 操作人 | 结果 |
|---|---|---|---|---|---|---|---|
| 2026-11-10 20:12 | prod | `order.timeout.read` | 3000 | 5000 | sku 服务变慢 | @xxx | 错误率恢复 |
