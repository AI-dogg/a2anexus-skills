# 订单流转、收货、错误码

主入口：[../SKILL.md](../SKILL.md)

## 订单状态流转

```
POST /orders → in_progress ──tasks/send──► … ──► receipt → completed
```

- 下单后状态为 `in_progress`（与 Hub `order_status` 一致）
- 任务与对话在 `in_progress` 下推进
- 调用收货接口 → `completed`，流程结束
- 同一 `orderId` 可多次 `tasks/send`（多轮对话）；上下文由 Hub 按订单维护。
- OpenAI 路由：`model` 在 **`POST .../register/openai`** 顶层填写并持久化；`tasks/send` 的 `message` 仅需用户输入，网关负责注入 `model` / `previous_response_id`（见 `92-REST` §2.0.1、§2.6）。

## `tasks/send` 响应

| 字段 | 说明 |
| --- | --- |
| `hubRequestId` | 网关追踪 ID |
| `agentId` | 下游 Agent UUID |
| `taskId` | 任务 ID |
| `state` | `completed` / `failed` |
| `completed` | 是否已结束 |
| `downstreamResponse` | 下游原始响应 JSON |

## 收货

手动确认收货须带评价字段（与 Hub 实现一致）：

```bash
curl -s -X POST "http://127.0.0.1:8080/api/v1/orders/{ORDER_ID}/receipt" \
  -H "Content-Type: application/json" \
  -d '{"requesterId":"hermes-test","rating":5,"comment":"ok"}'
```

订单状态变为 `completed`。

## 错误码

### Gateway

| 代码 | HTTP | 含义 |
| --- | --- | --- |
| `agent_route_not_found` | 404 | `agentId` 不存在或未激活 |
| `order_not_found` | 404 | `orderId` 不存在 |
| `order_agent_mismatch` | 409 | `orderId` 不属于该 `agentId` |
| `order_not_executable` | 409 | 订单状态不可执行 |
| `order_not_awaiting_receipt` | 409 | 订单不在可收货状态 |
| `downstream_http_error` | 502 | 下游返回 HTTP 错误 |
| 422 | 422 | 请求体校验失败 |

### 下游（Hermes API Server）

| 现象 | 原因 |
| --- | --- |
| `-32601 Method not found` | `baseUrl` 指向了 A2A 端口（8081） |
| `400 Missing 'messages' field` | `openaiResponsesPath` 设成了 `/v1/chat/completions` |
| `401 Invalid API key` | 没带 `bearerToken` 或 key 错误 |
