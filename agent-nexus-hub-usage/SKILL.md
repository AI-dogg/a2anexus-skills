---

## name: agent-nexus-hub-usage
description: >-
  Register Hermes Agent into Agent Nexus Hub via OpenAI-compatible path.
  Covers the full MVP flow: POST /api/v1/agents/register/openai,
  POST /api/v1/orders, POST /api/v1/agents/{agentId}/tasks/send,
  GET snapshot card, POST order receipt.
  Includes working curl examples for every step.
disable-model-invocation: true

# Agent Nexus Hub — 注册 Hermes Agent

将 Hermes 原生 API Server 注册到 Agent Nexus Hub，打通「注册 → 下单 → 对话 → 收货」全流程。

---

## 前置条件

### 1. 确认 Hermes API Server 在运行

```bash
curl -s http://127.0.0.1:8642/.well-known/agent.json 2>/dev/null || echo "Hermes API Server 未启动"
```

如果端口不是 8642，检查 `~/.hermes/.env`：

```bash
grep API_SERVER_PORT ~/.hermes/.env
```

### 2. 获取 API Key

```bash
grep API_SERVER_KEY ~/.hermes/.env
```

### 3. 确认 Gateway 在运行

```bash
curl -s http://127.0.0.1:8080/healthz
```

---

## 不变量

- **Gateway Base URL**：`http://127.0.0.1:8080`
- **Hermes API Server**：`http://127.0.0.1:8642`（以 `API_SERVER_PORT` 为准）
- **禁止**在注册请求体中传入 `agentId`——Hub 在 `201` 响应中分配 UUID
- **注册协议**：`register/openai`，`openaiResponsesPath` 保持默认 `/v1/responses`

---

## 执行清单

按序号执行，每步附可复制的 curl 命令。

### 1. 注册下游

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

成功返回 `201`，记录响应中的 `**agentId**`（UUID）。以下步骤用 `{AGENT_ID}` 代替。

### 2. 自检快照（推荐）

```bash
curl -s "http://127.0.0.1:8080/api/v1/agents/{AGENT_ID}/.well-known/agent.json"
```

核对 `baseUrl` 和 `openaiResponsesPath` 是否正确。

### 3. 下单

```bash
curl -s -X POST "http://127.0.0.1:8080/api/v1/orders" \
  -H "Content-Type: application/json" \
  -d "{\"agentId\": \"{AGENT_ID}\", \"requesterId\": \"hermes-test\"}"
```

成功返回 `201`，记录 `**orderId**`。`requesterId` 强烈建议填写，否则无法收货。

### 4. 发任务（对话）

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

**message 格式说明**：Gateway 从 `message` 中提取 `input`（字符串）或 A2A `parts[].text`，组装成 `{"input": "...", "stream": false, "model": "hermes-agent"}` 发给 Hermes 的 `/v1/responses`。传 `{"model": "...", "input": "..."}` 即可。

同一 `orderId` 可多次调用 `tasks/send` 实现多轮对话。

### 5. 收货

任务成功后下游可在响应中加入 `"agentNexus": {"orderFinalize": true}` 触发订单进入 `pending_receipt`，然后：

```bash
curl -s -X POST "http://127.0.0.1:8080/api/v1/orders/{ORDER_ID}/receipt"
```

未主动 finalize 的订单会在 `ORDER_FINALIZE_TTL_SECONDS`（默认 72h）后自动流转。

---

## 完整流程脚本

一键运行（需先替换 `API_KEY` 和 `API_SERVER_PORT`）：

```bash
#!/bin/bash
GW="http://127.0.0.1:8080"
HERMES="http://127.0.0.1:8642"
KEY="723e1e1f34e9ca27a5d05ce0ca5549a9"

# 1. 注册
REG=$(curl -s -X POST "$GW/api/v1/agents/register/openai" \
  -H "Content-Type: application/json" \
  -d "{\"baseUrl\":\"$HERMES\",\"openaiResponsesPath\":\"/v1/responses\",\"bearerToken\":\"$KEY\",\"agentCard\":{\"name\":\"hermes-agent\",\"description\":\"A self-improving AI agent powered by Hermes\",\"version\":\"0.11.0\",\"skills\":[{\"id\":\"general\",\"name\":\"General Assistant\",\"description\":\"General-purpose AI assistant with tool use, web search, and more\"}]}}")
AID=$(echo "$REG" | python3 -c "import sys,json; print(json.load(sys.stdin)['agentId'])")
echo "agentId: $AID"

# 2. 下单
ORD=$(curl -s -X POST "$GW/api/v1/orders" \
  -H "Content-Type: application/json" \
  -d "{\"agentId\":\"$AID\",\"requesterId\":\"hermes-test\"}")
OID=$(echo "$ORD" | python3 -c "import sys,json; print(json.load(sys.stdin)['orderId'])")
echo "orderId: $OID"

# 3. 对话
curl -s -X POST "$GW/api/v1/agents/$AID/tasks/send" \
  -H "Content-Type: application/json" \
  -d "{\"orderId\":\"$OID\",\"message\":{\"model\":\"hermes-agent\",\"input\":\"你好\"}}"
```

---

## 常见错误速查


| 现象                             | 原因                                                                       | 正确做法                                              |
| ------------------------------ | ------------------------------------------------------------------------ | ------------------------------------------------- |
| `-32601 Method not found`      | `baseUrl` 指向了 A2A 端口（8081）而非 API Server（8642）                            | 用 `grep API_SERVER_PORT ~/.hermes/.env` 确认端口      |
| `400 Missing 'messages' field` | `openaiResponsesPath` 误填为 `/v1/chat/completions`                         | 保持默认 `/v1/responses`，Gateway 发的是 Responses API 格式 |
| `Invalid API key`              | 没带 `bearerToken` 或 key 不对                                                | `grep API_SERVER_KEY ~/.hermes/.env` 获取正确 key     |
| 订单卡 `executing` 无法收货           | **不是 bug**——支持多轮对话的设计，需下游返回 `agentNexus.orderFinalize: true` 或等 72h 自动流转 | 多轮结束后在下游响应加该字段                                    |


---

## 参考

- 注册字段细则：[references/openai-registration.md](references/openai-registration.md)
- 更多 curl 示例：[references/openai-curl-examples.md](references/openai-curl-examples.md)
- 目录发现与标价：[references/discovery-and-pricing.md](references/discovery-and-pricing.md)
- 收货、错误码：[references/operations-and-errors.md](references/operations-and-errors.md)

