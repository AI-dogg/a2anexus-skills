# 订单流转、收货、错误码

主入口：[../SKILL.md](../SKILL.md)

## 订单状态流转

```
created → in_progress → completed
                ↑
          tasks/send 后进入
                │
                ▼
         POST .../receipt → completed
```

手动收货须带 JSON body（`requesterId` + `rating` + `comment`）才会落库评价；Redis/TTL 等**自动**收货路径不写评价记录。

- 下单后状态为 `created`
- 首次 `tasks/send` 成功后进入 `in_progress`
- 调用收货接口 → `completed`，流程结束
- 同一 `orderId` 可多次 `tasks/send`（多轮对话）

## `tasks/send` 响应


| 字段                   | 说明                     |
| -------------------- | ---------------------- |
| `hubRequestId`       | 网关追踪 ID                |
| `agentId`            | 下游 Agent UUID          |
| `taskId`             | 任务 ID                  |
| `state`              | `completed` / `failed` |
| `completed`          | 是否已结束                  |
| `downstreamResponse` | 下游原始响应 JSON            |


## 收货

```bash
curl -s -X POST "http://127.0.0.1:8080/api/v1/orders/{ORDER_ID}/receipt" \
  -H "Content-Type: application/json" \
  -d '{"requesterId":"hermes-test","rating":5,"comment":"很好用"}'
```

订单状态变为 `completed`；带 body 时同时写入一条订单评价。字段、情感阈值与边界见 [reviews-and-sentiment.md](reviews-and-sentiment.md)。

## 错误码

### Gateway


| 代码                            | HTTP | 含义                                     |
| ----------------------------- | ---- | -------------------------------------- |
| `agent_route_not_found`       | 404  | `agentId` 不存在或未激活                      |
| `order_not_found`             | 404  | `orderId` 不存在                          |
| `order_agent_mismatch`        | 409  | `orderId` 不属于该 `agentId`               |
| `order_not_executable`        | 409  | 订单状态不可执行                               |
| `order_not_awaiting_receipt`  | 409  | 订单不在可收货状态                              |
| `receipt_review_required`     | 422  | 手动收货缺 `requesterId`/`rating`/`comment` |
| `receipt_requester_mismatch`  | 403  | body 中 `requesterId` 与订单不一致            |
| `order_review_already_exists` | 409  | 该订单已有评价，重复提交收货                         |
| `order_review_not_found`      | 404  | 该订单尚无评价记录                              |
| `downstream_http_error`       | 502  | 下游返回 HTTP 错误                           |
| 422                           | 422  | 请求体校验失败                                |


### 下游（Hermes API Server）


| 现象                             | 原因                                               |
| ------------------------------ | ------------------------------------------------ |
| `-32601 Method not found`      | `baseUrl` 指向了 A2A 端口（8081）                       |
| `400 Missing 'messages' field` | `openaiResponsesPath` 设成了 `/v1/chat/completions` |
| `401 Invalid API key`          | 没带 `bearerToken` 或 key 错误                        |


