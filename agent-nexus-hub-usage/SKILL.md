---
name: agent-nexus-hub-usage
description: >-
  Guides integration via Agent Nexus Gateway MVP for OpenAI-compatible downstream agents:
  POST /api/v1/agents/register/openai, persist server-assigned agentId (UUID), POST /api/v1/orders,
  POST /api/v1/agents/{agentId}/tasks/send with required orderId and message or jsonrpc,
  optional GET snapshot card, POST order receipt. Hub, Gateway, receipt, OpenAI registration.
  Core checklist in SKILL.md; registration fields, curl, discovery/pricing, receipt/errors in references/.
  Does not document A2A paths (register/a2a, POST …/a2a, tasks polling) in the MVP narrative—see
  gateway/src/api/routes.py and Vault Gateway 92-REST when required.
disable-model-invocation: true
---

# Agent Nexus Hub（MVP｜OpenAI 下游）

实现以仓库 **`gateway/src/api/routes.py`** 为准。本文件是 **单一执行清单**：按序号做完即可打通「注册 → 下单 → 调用 → 收货」。

## 本期不包含（避免与 MVP 混淆）

**本 Skill 正文不展开** `POST /api/v1/agents/register/a2a`、`POST /api/v1/agents/{agentId}/a2a`、以及 **`GET /api/v1/a2a/tasks/{hubRequestId}`** 轮询等 A2A 专项流程；网关仍可能有这些路由，需要时直接读源码与 Vault `开发文档/Gateway/92-REST.md`。

---

## 渐进式：何时打开 references

| 场景 | 文件 |
| --- | --- |
| 填 `register/openai` 请求体 / `agentCard` | [references/openai-registration.md](references/openai-registration.md) |
| 可复制 curl | [references/openai-curl-examples.md](references/openai-curl-examples.md) |
| 目录发现、标价 | [references/discovery-and-pricing.md](references/discovery-and-pricing.md) |
| 收货语义、自动收货、`hubRequestId`、错误码 | [references/operations-and-errors.md](references/operations-and-errors.md) |

更长叙述（Obsidian）：`Projects/AgentNexus/开发文档/Gateway/92-REST.md`、`95-注册存根.md`。可选：`scripts/`、`assets/` 占位目录，按需扩展。

---

## 不变量

1. **Base URL**：`http://{GATEWAY_HOST}:{GATEWAY_PORT}`（默认 `127.0.0.1:8080`）。
2. **禁止**在注册请求体中传入 `agentId`：**Hub 在 `201` 响应 JSON 中分配 `agentId`（UUID）**，须持久化；此后路径中的 `{agentId}` 均使用该值。
3. **鉴权**：当前管理面多为开放 API；生产应为 `requesterId`、收货等接入身份方案。

---

## Hub `agentId` 占位（注册成功后回填）

完成下方步骤 **1** 且拿到 **`201` 响应后**，把 JSON 里的 **`agentId`（UUID）** 填进本 Skill（或同步到 Cursor Rule / 环境变量 / 密钥库）。**后续凡写 `{agentId}`，均替换为该 UUID。**

| 占位 | 填写 |
| --- | --- |
| **`HUB_AGENT_ID`** | `<粘贴 201 响应中的 agentId>` |

---

## MVP 执行清单（OpenAI）

1. **注册下游**  
   `POST /api/v1/agents/register/openai`  
   - 必填：`baseUrl`、嵌套 **`agentCard`**（细则见 [references/openai-registration.md](references/openai-registration.md)）。  
   - 可选：`openaiResponsesPath`（默认 `/v1/responses`）、`bearerToken`、`quoteAmount` / `quoteCurrency`。  
   - 成功 **`201`** → 读取响应体 **`agentId`**，并**回填**到上文 **`HUB_AGENT_ID`** 占位。

2. **（推荐）自检快照**  
   `GET /api/v1/agents/{agentId}/.well-known/agent.json`（`{agentId}` = **`HUB_AGENT_ID`**）  
   - 返回注册时落库的 Card **快照**（非每次实时拉下游）。

3. **下单**  
   `POST /api/v1/orders`  
   - 必填：body 中 **`agentId`** = **`HUB_AGENT_ID`**。  
   - **强烈建议**：`requesterId`（无则收货闭环可能失败，见 [references/operations-and-errors.md](references/operations-and-errors.md)）。  
   - 成功 **`201`** → 保存 **`orderId`**。

4. **发任务（调用下游）**  
   `POST /api/v1/agents/{agentId}/tasks/send`（路径中 `{agentId}` = **`HUB_AGENT_ID`**）  
   - **必填**：`orderId`（须对应同一 `agentId` 且订单可执行）；**`message`** 与 **`jsonrpc`** 二选一必填（不可都缺）。  
   - 可选：`taskId`；请求头可传 **`x-hub-request-id`** 与响应 **`hubRequestId`** 对齐。  
   - 网关将 OpenAI 协议请求映射为下游 **`POST {baseUrl}{openaiResponsesPath}`**（如 `/v1/responses`），与注册时 `bearerToken` 等一致。

5. **订单与收货**  
   - 任务成功结束后订单进入 **`pending_receipt`**（通常需订单含 `requesterId`）；详见 [references/operations-and-errors.md](references/operations-and-errors.md)。  
   - **确认收货**：`POST /api/v1/orders/{orderId}/receipt`。  
   - 超时自动收货依赖 Redis 与配置（默认约 72h），见同 reference。

---

## 相关入口 Skill

- **`register-openai-agent`**：关键词入口，正文指向本目录与 **references/openai-*.md**。
