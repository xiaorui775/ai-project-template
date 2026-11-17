# 接口文档（示例）

> 作用：前后端与 AI 共享的接口契约。改接口三件套：`后端 DTO` → `本文件` → `packages/shared-types`。

## Invoice 发票

### POST `/api/v1/invoices/batch-download`

批量打包下载发票 PDF。

**权限**：`invoice:download`

**请求头**

| Header | 必填 | 说明 |
|---|---|---|
| `Authorization` | 是 | `Bearer <token>` |
| `X-Trace-Id` | 否 | 不传由网关生成 |

**请求体**

```json
{ "ids": ["1893456789012345678", "1893456789012345679"] }
```

| 字段 | 类型 | 必填 | 约束 |
|---|---|---|---|
| ids | string[] | 是 | 1–50 个，雪花 ID 用 string |

**响应**

- 成功：`200`，`Content-Type: application/zip`，文件 `invoices-20261117.zip`
- 失败：`Result<Void>`，`code` 见下表

| code | msg | 触发条件 |
|---|---|---|
| 20403 | 无权限 | 缺少 `invoice:download` |
| 20422 | 单次最多下载 50 张 | ids.length > 50 |
| 1 | 服务繁忙，请稍后重试 | 下游异常降级 |

**幂等**：幂等。重复下载不影响数据。

---

### GET `/api/v1/invoices`

分页查询。

| Query | 类型 | 说明 |
|---|---|---|
| page | int | 1 基，默认 1 |
| pageSize | int | 默认 20，最大 100 |
| month | string | `YYYY-MM`，可选 |
| status | string | `UNISSUED` / `ISSUING` / `ISSUED` / `VOID` |

响应 `Result<PageResult<InvoiceVO>>`：

```json
{
  "code": 0,
  "msg": "ok",
  "data": {
    "total": 128,
    "list": [
      { "id": "1893...", "invoiceNo": "043002100511", "amount": "1280.00", "status": "ISSUED", "issuedAt": "2026-11-01T10:00:00" }
    ]
  },
  "traceId": "9f1c..."
}
```

**注意**：`amount` 为 string（BigDecimal 序列化），`id` 为 string（雪花 ID）。
