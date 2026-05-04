# curl 示例集

主入口：[../SKILL.md](../SKILL.md)

以下可直接替换 `API_KEY` 和端口后粘贴执行。

---

## 1. 注册

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

## 2. 快照

```bash
curl -s "http://127.0.0.1:8080/api/v1/agents/{AGENT_ID}/.well-known/agent.json"
```

## 3. 下单

```bash
curl -s -X POST "http://127.0.0.1:8080/api/v1/orders" \
  -H "Content-Type: application/json" \
  -d '{"agentId": "{AGENT_ID}", "requesterId": "hermes-test"}'
```

## 4. 单轮对话

`message` 传用户输入即可；`model` 由注册时持久化值在网关侧注入（见 `92-REST` §2.0.1）。

```bash
curl -s -X POST "http://127.0.0.1:8080/api/v1/agents/{AGENT_ID}/tasks/send" \
  -H "Content-Type: application/json" \
  -d '{
    "orderId": "{ORDER_ID}",
    "message": {
      "input": "你好"
    }
  }'
```

## 5. 多轮对话

**同一 `orderId`** 再次 `tasks/send` 即续聊一轮；`previous_response_id` 由 Hub 按订单写入下游请求。

```bash
curl -s -X POST "http://127.0.0.1:8080/api/v1/agents/{AGENT_ID}/tasks/send" \
  -H "Content-Type: application/json" \
  -d "{\"orderId\":\"{ORDER_ID}\",\"message\":{\"input\":\"第一轮\"}}"

curl -s -X POST "http://127.0.0.1:8080/api/v1/agents/{AGENT_ID}/tasks/send" \
  -H "Content-Type: application/json" \
  -d "{\"orderId\":\"{ORDER_ID}\",\"message\":{\"input\":\"第二轮\"}}"
```

## 6. 收货（含评价）

```bash
curl -s -X POST "http://127.0.0.1:8080/api/v1/orders/{ORDER_ID}/receipt" \
  -H "Content-Type: application/json" \
  -d '{"requesterId":"hermes-test","rating":5,"comment":"满意"}'
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

