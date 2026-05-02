# 订单收尾、hubRequestId、错误码（MVP 补充）

主入口：[../SKILL.md](../SKILL.md)

## 与 MVP 主干的关系

上文 Skill 清单覆盖 **注册 → 下单 → `tasks/send`**。本节说明 **响应里的 `hubRequestId`**、**订单收货**与常见 **Gateway 错误码**，避免 Agent 漏步骤或误判终态。

## `tasks/send` 响应（TaskPollingResponse）

- 响应体含 **`hubRequestId`**（camelCase）、**`agentId`**、**`taskId`**、**`state`**、**`completed`**、**`downstreamResponse`** 等（以 `gateway/src/api/schemas.py` 为准）。
- **`hubRequestId`**：可与请求头 **`x-hub-request-id`** 对齐，用于日志与追踪。
- **MVP（OpenAI）**：多为短路径返回；若业务上出现长运行或需轮询，网关上另有 **`GET /api/v1/a2a/tasks/{hubRequestId}`**（不在本 MVP Skill 展开；见 `routes.py`）。

## 收货与订单状态（摘要）

- 任务**成功结束**后，订单可进入 **`pending_receipt`**（通常需要下单时提供 **`requesterId`**；否则可能进入 **`failed`** 等态，以实现为准）。
- **主动收货**：`POST /api/v1/orders/{orderId}/receipt`。
- **未收货超时**：默认约 **72h**（`RECEIPT_TTL_SECONDS` 等）；依赖 Redis 与惰性/监听逻辑；详见仓库 `.env.example` 与 Vault `93-存储`。
- 读单笔订单：`GET /api/v1/orders/{orderId}` 可能触发惰性自动收货（以实现为准）。

## 错误码（摘录）

| 代码 / HTTP | 含义（典型） |
| --- | --- |
| `agent_route_not_found` | 404，未知或未激活的 `agentId` |
| `order_not_found` | 404 |
| `order_agent_mismatch` / `order_not_executable` | 409，`orderId` 与路径 `agentId` 不匹配或订单不可执行 |
| `receipt_requester_required` | 409，收货需要 `requesterId` 等前置条件不满足 |
| **422** | 请求体验证失败（例如 **`tasks/send` 缺少 `orderId`**，或 **`message`/`jsonrpc` 均未提供**） |
