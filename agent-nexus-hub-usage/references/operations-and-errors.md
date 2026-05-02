# 订单流转、收货、错误码

主入口：[../SKILL.md](../SKILL.md)

## `tasks/send` 响应

响应体字段（以 `gateway/src/api/schemas.py` 为准）：


| 字段                   | 说明                                  |
| -------------------- | ----------------------------------- |
| `hubRequestId`       | 网关追踪 ID，可与请求头 `x-hub-request-id` 对齐 |
| `agentId`            | 下游 Agent 的 Hub UUID                 |
| `taskId`             | 本次任务 ID                             |
| `state`              | 任务状态（`completed` / `failed` 等）      |
| `completed`          | 是否已结束                               |
| `downstreamResponse` | 下游的原始响应 JSON                        |


## 订单状态流转

```
created → executing → pending_receipt → completed
                         ↑                    ↑
                    需 orderFinalize     POST .../receipt
                    或 72h TTL 到期
```

- `**executing**`：默认状态，支持同一 `orderId` 多次 `tasks/send`（多轮对话）。
- **进入 `pending_receipt`** 的条件（任一满足）：
  1. 下游响应中包含 `"agentNexus": {"orderFinalize": true}` — Agent 主动声明可结算
  2. `ORDER_FINALIZE_TTL_SECONDS`（默认 72h）到期 — 兜底自动流转
- **前提**：下单时需提供 `requesterId`，否则流转可能失败。

## 收货

**主动收货**：

```bash
curl -s -X POST "http://127.0.0.1:8080/api/v1/orders/{ORDER_ID}/receipt"
```

订单必须在 `pending_receipt` 状态才能收货。`executing` 状态下调用会返回：

```json
{"detail": {"code": "order_not_awaiting_receipt", "message": "Order is not awaiting receipt."}}
```

**自动收货**：进入 `pending_receipt` 后，`RECEIPT_TTL_SECONDS`（默认 72h）到期自动完成，依赖 Redis。

## 错误码

### Gateway 层


| 代码                           | HTTP | 含义                                              |
| ---------------------------- | ---- | ----------------------------------------------- |
| `agent_route_not_found`      | 404  | `agentId` 不存在或未激活                               |
| `order_not_found`            | 404  | `orderId` 不存在                                   |
| `order_agent_mismatch`       | 409  | `orderId` 不属于该 `agentId`                        |
| `order_not_executable`       | 409  | 订单不在 `created` 或 `executing` 状态                 |
| `order_not_awaiting_receipt` | 409  | 订单不在 `pending_receipt` 状态，无法收货                  |
| `downstream_http_error`      | 502  | 下游返回 HTTP 错误（400/401/500 等）                     |
| `downstream_timeout`         | 504  | 下游超时                                            |
| `downstream_unreachable`     | 502  | 下游不可达                                           |
| 422                          | 422  | 请求体校验失败（如缺 `orderId`、`message` 和 `jsonrpc` 都没填） |


### 下游层（Hermes API Server）


| 现象                                        | 原因                                                                  |
| ----------------------------------------- | ------------------------------------------------------------------- |
| `-32601 Method not found`                 | `baseUrl` 指向了 A2A 端口（8081）而非 API Server（8642）                       |
| 400 `Missing or invalid 'messages' field` | `openaiResponsesPath` 设成了 `/v1/chat/completions`，应为 `/v1/responses` |
| 401 `Invalid API key`                     | 注册时没带 `bearerToken` 或 key 过期                                        |


### Gateway 500

如果订单接口返回 500 `Internal Server Error`，通常是 Gateway 后端依赖（DB/Redis）未就绪，重试或检查 Gateway 日志。