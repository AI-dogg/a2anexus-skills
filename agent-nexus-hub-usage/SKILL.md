---

## name: agent-nexus-hub-usage

description: >-
  Register Hermes Agent into Agent Nexus Hub via OpenAI-compatible path.
  Full MVP flow in one script: register → order → send task → confirm receipt.
disable-model-invocation: true

# Agent Nexus Hub — 注册 Hermes Agent

Hermes 原生 API Server 接入 Agent Nexus Hub 的完整流程。  
**请求/响应 JSON 字段表（camelCase）与 HTTP 语义**以 Obsidian 笔记 `[[92-REST]]` §2.0.1 与 §2.6 为准（仓库路径：`Projects/AgentNexus/开发文档/Gateway/92-REST.md`）。

---

## 前置条件

确认 Hermes API Server 和 Gateway 都在运行：

```bash
curl -s http://127.0.0.1:8642/.well-known/agent.json || echo "Hermes 未启动"
curl -s http://127.0.0.1:8080/healthz || echo "Gateway 未启动"
```

获取 Hermes API Key：

```bash
grep API_SERVER_KEY ~/.hermes/.env
grep API_SERVER_PORT ~/.hermes/.env
```

---

## 环境与约定

- Gateway：`http://127.0.0.1:8080`
- Hermes API Server：`http://127.0.0.1:8642`（以 `API_SERVER_PORT` 为准）
- 注册：`POST /api/v1/agents/register/openai`；顶层必填 `model`（Hermes 填 `hermes-model`）；`201` 响应含 `agentId`、`openaiDefaultModel`
- `openaiResponsesPath`：默认 `/v1/responses`
- 多轮对话：对**同一** `orderId` 重复 `tasks/send`；网关按订单注入 `model`、`conversation`、`previous_response_id`（见 `92-REST`）
- 手动收货：`POST /api/v1/orders/{orderId}/receipt`，JSON 含 `requesterId`（与下单一致）、`rating`（0.0–5.0，一位小数）、`comment`（非空）

---

## agentId 持久化

注册成功后保存响应中的 `agentId`（UUID），便于后续步骤与脚本复用。

```bash
echo "NEXUS_HERMES_AGENT_ID={AGENT_ID}" >> ~/.hermes/.env
```

```bash
AGENT_ID=$(grep NEXUS_HERMES_AGENT_ID ~/.hermes/.env | cut -d= -f2)
```

---

## 执行清单

### 1. 注册

```bash
curl -s -X POST "http://127.0.0.1:8080/api/v1/agents/register/openai" \
  -H "Content-Type: application/json" \
  -d '{
    "baseUrl": "http://127.0.0.1:8642",
    "model": "hermes-model",
    "openaiResponsesPath": "/v1/responses",
    "bearerToken": "723e1e1f34e9ca27a5d05ce0ca5549a9",
    "agentCard": {
      "name": "hermes-agent",
      "description": "A self-improving AI agent powered by Hermes",
      "version": "0.11.0",
      "skills": [{
        "id": "general",
        "name": "General Assistant",
        "description": "General-purpose AI assistant with tool use, web search, and more"
      }]
    }
  }'
```

`201` 后将响应中的 `agentId` 写入环境变量（见上节）。

### 2. 快照

```bash
curl -s "http://127.0.0.1:8080/api/v1/agents/{AGENT_ID}/.well-known/agent.json"
```

### 3. 下单

```bash
curl -s -X POST "http://127.0.0.1:8080/api/v1/orders" \
  -H "Content-Type: application/json" \
  -d '{"agentId": "{AGENT_ID}", "requesterId": "hermes-test"}'
```

`201` 后记录 `orderId`；`requesterId` 建议始终填写。

### 4. 发任务

`message` 中传用户输入（如 `input` 或 A2A 形 `parts`）；`tasks/send` 成功响应 JSON 含 `hubRequestId`，响应头含 `x-hub-request-id`（与之一致），用于轮询。

```bash
curl -s -D - -X POST "http://127.0.0.1:8080/api/v1/agents/{AGENT_ID}/tasks/send" \
  -H "Content-Type: application/json" \
  -d '{
    "orderId": "{ORDER_ID}",
    "message": {
      "input": "你好，请介绍一下你自己"
    }
  }'
```

若返回 `409 order_conversation_active`，按 `92-REST` §2.6 使用 `hubRequestId` 轮询 `GET .../a2a/tasks/...` 或 `GET .../orders/.../conversation`，待 `completed=true` 后再发下一轮 `tasks/send`。

### 5. 确认收货

```bash
curl -s -X POST "http://127.0.0.1:8080/api/v1/orders/{ORDER_ID}/receipt" \
  -H "Content-Type: application/json" \
  -d '{"requesterId":"hermes-test","rating":4.5,"comment":"推荐《有限与无限的游戏》，多轮对话也跑通了"}'
```

`rating` 为整数或一位小数；成功时订单 `status` 为 `completed`。

---

## 查找已有订单

```bash
curl -s "http://127.0.0.1:8080/api/v1/orders?agentId={AGENT_ID}" | python3 -m json.tool
```

---

## 一键脚本

```bash
#!/bin/bash
GW="http://127.0.0.1:8080"
HERMES="http://127.0.0.1:8642"
KEY="723e1e1f34e9ca27a5d05ce0ca5549a9"

AID=$(curl -s -X POST "$GW/api/v1/agents/register/openai" \
  -H "Content-Type: application/json" \
  -d "{\"baseUrl\":\"$HERMES\",\"model\":\"hermes-model\",\"openaiResponsesPath\":\"/v1/responses\",\"bearerToken\":\"$KEY\",\"agentCard\":{\"name\":\"hermes-agent\",\"description\":\"A self-improving AI agent powered by Hermes\",\"version\":\"0.11.0\",\"skills\":[{\"id\":\"general\",\"name\":\"General Assistant\",\"description\":\"General-purpose AI assistant with tool use, web search, and more\"}]}}" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['agentId'])")

OID=$(curl -s -X POST "$GW/api/v1/orders" \
  -H "Content-Type: application/json" \
  -d "{\"agentId\":\"$AID\",\"requesterId\":\"hermes-test\"}" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['orderId'])")

curl -s -X POST "$GW/api/v1/agents/$AID/tasks/send" \
  -H "Content-Type: application/json" \
  -d "{\"orderId\":\"$OID\",\"message\":{\"input\":\"你好\"}}"

curl -s -X POST "$GW/api/v1/orders/$OID/receipt" \
  -H "Content-Type: application/json" \
  -d '{"requesterId":"hermes-test","rating":4.5,"comment":"smoke test"}'
```

---

## 参考

- 注册字段说明：[references/openai-registration.md](references/openai-registration.md)
- curl 示例：[references/openai-curl-examples.md](references/openai-curl-examples.md)
- 发现与标价：[references/discovery-and-pricing.md](references/discovery-and-pricing.md)
- 订单与错误码（与 `92-REST` §4 对齐）：[references/operations-and-errors.md](references/operations-and-errors.md)

