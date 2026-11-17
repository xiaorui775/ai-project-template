# Runbook：服务不可用

## 判定

| 信号 | 含义 |
|---|---|
| Sentinel `block` 指标突增 | 触发限流，可能是流量突增或下游变慢 |
| Feign fallback 日志 `warn` 激增 | 下游不可用或超时 |
| 网关 502/504 比例 > 1% | 后端实例不可用 |
| Nacos 实例数 < 期望值 | 实例掉线/被摘除 |

## 处置顺序

1. **确认影响面**：`kubectl get pods -n <env>` 或 Nacos 控制台服务列表看实例数与健康状态。
2. **看最近变更**：`git log --oneline -10`、Nacos 配置历史（有无刚改的配置）。
3. **止血优先于定位**：
   - 若是新版本发布 → 立即回滚（Nacos 权重调 0 或回滚 Deployment）。
   - 若是流量突增 → Sentinel 限流阈值临时下调，保护后端。
   - 若是下游服务挂 → 确认该服务 fallback 是否生效，必要时手动降级开关。
4. **保留现场**：导出日志、JVM 线程栈（`jstack`）、`heap`（若疑似 OOM）。
5. **恢复后**：补写 `docs/known-pitfalls.md` 一条 + 相关告警规则。

## 常用命令

```bash
# 查看某个服务所有实例
curl -s "http://nacos:8848/nacos/v1/ns/instance/list?serviceName=order-service&namespaceId=prod"

# 查看网关路由是否生效
curl -s http://gateway:8080/actuator/gateway/routes | jq '.[].uri'

# 健康检查
curl -s http://order-service:8081/actuator/health | jq
```

## 回滚

```bash
# 后端：回滚到上一个镜像
kubectl -n prod rollout undo deployment/order-service
# 前端：静态资源回滚（保留上一版本目录，nginx 改软链）
ln -sfn /srv/web/previous /srv/web/current && nginx -s reload
```

## 禁止

- 禁止在生产直接改数据库表结构（走 Flyway 新版本）。
- 禁止在未保留现场的情况下重启并清空日志。