# Changelog

格式遵循 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/)，版本号遵循 SemVer。

## [Unreleased]

### Added
- 发票批量下载接口 `POST /api/v1/invoices/batch-download`（REQ-001）

## [web-v1.4.0] - 2026-11-01

### Added
- 订单列表新增按商品名搜索

### Changed
- **BREAKING**: 所有雪花 ID 字段由 `number` 改为 `string`
  - 迁移：前端类型取 `@repo/shared-types`，二次转换的地方用 `String(id)`

## [svc-order-v2.1.3] - 2026-10-28

### Fixed
- 创建订单时库存扣减失败后订单仍落库（P-06）
