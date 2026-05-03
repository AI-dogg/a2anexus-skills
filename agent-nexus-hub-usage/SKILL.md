---

## name: agent-nexus-hub-usage
description: >-
  Register Hermes Agent into Agent Nexus Hub via OpenAI-compatible path.
  Full MVP flow in one script: register → order → send task → confirm receipt.
disable-model-invocation: true

# Agent Nexus Hub — 注册 Hermes Agent

Hermes 原生 API Server 接入 Agent Nexus Hub 的完整流程。

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

## 不变量

- Gateway：`http://127.0.0.1:8080`
- Hermes API Server：`http://127.0.0.1:8642`（以 `API_SERVER_PORT` 为准）
- 注册协议：`POST /api/v1/agents/register/openai`
- `openaiResponsesPath`：`/v1/responses`（默认值，不需要改）
- 禁止在注册请求中传入 `agentId`，Hub 分配 UUID

---

## agentId 持久化

注册成功后 Hub 返回 `agentId`（UUID），**必须持久化到本地**，否则 skill 更新后会丢失。

```bash
echo "NEXUS_HERMES_AGENT_ID={AGENT_ID}" >> ~/.hermes/.env
```

后续步骤中的 `{AGENT_ID}` 均替换为该值。也可以直接从环境变量读取：

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

返回 201，将响应中的 `agentId` 持久化：

```bash
echo "NEXUS_HERMES_AGENT_ID=<粘贴 agentId>" >> ~/.hermes/.env
```

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

返回 201，记录响应中的 `orderId`。`requesterId` 必填。

### 4. 发任务

```bash
curl -s -X POST "http://127.0.0.1:8080/api/v1/agents/{AGENT_ID}/tasks/send" \
  -H "Content-Type: application/json" \
  -d '{
    "orderId": "{ORDER_ID}",
    "message": {
      "model": "hermes-agent",
      "input": "你好，请介绍一下你自己"
    }
  }'
```

同一 `orderId` 可多次调用实现多轮对话。Gateway 将 message 转为 `{"input": "...", "stream": false, "model": "..."}` 发给 Hermes 的 `/v1/responses`。

### 5. 确认收货

```bash
curl -s -X POST "http://127.0.0.1:8080/api/v1/orders/{ORDER_ID}/receipt"
```

订单状态变为 `completed`，流程结束。

---

## 查找已有订单

通过 agentId 查找订单列表：

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
  -d "{\"baseUrl\":\"$HERMES\",\"openaiResponsesPath\":\"/v1/responses\",\"bearerToken\":\"$KEY\",\"agentCard\":{\"name\":\"hermes-agent\",\"description\":\"A self-improving AI agent powered by Hermes\",\"version\":\"0.11.0\",\"skills\":[{\"id\":\"general\",\"name\":\"General Assistant\",\"description\":\"General-purpose AI assistant with tool use, web search, and more\"}]}}" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['agentId'])")

OID=$(curl -s -X POST "$GW/api/v1/orders" \
  -H "Content-Type: application/json" \
  -d "{\"agentId\":\"$AID\",\"requesterId\":\"hermes-test\"}" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['orderId'])")

curl -s -X POST "$GW/api/v1/agents/$AID/tasks/send" \
  -H "Content-Type: application/json" \
  -d "{\"orderId\":\"$OID\",\"message\":{\"model\":\"hermes-agent\",\"input\":\"你好\"}}"

curl -s -X POST "$GW/api/v1/orders/$OID/receipt"
```

---

## 常见错误


| 现象                             | 原因                                              | 正确做法                                  |
| ------------------------------ | ----------------------------------------------- | ------------------------------------- |
| `-32601 Method not found`      | `baseUrl` 指向了 A2A 端口（8081）                      | 用 8642（API Server 端口）                 |
| `400 Missing 'messages' field` | `openaiResponsesPath` 写了 `/v1/chat/completions` | 保持默认 `/v1/responses`                  |
| `Invalid API key`              | 没带 `bearerToken`                                | 从 `~/.hermes/.env` 取 `API_SERVER_KEY` |
| `order_not_awaiting_receipt`   | 订单未进入可收货状态                                      | 确认下游任务已完成后再收货                         |


---

## 参考

- 注册字段详情：[references/openai-registration.md](references/openai-registration.md)
- curl 示例集：[references/openai-curl-examples.md](references/openai-curl-examples.md)
- 发现与标价：[references/discovery-and-pricing.md](references/discovery-and-pricing.md)
- 订单流转与错误码：[references/operations-and-errors.md](references/operations-and-errors.md)

