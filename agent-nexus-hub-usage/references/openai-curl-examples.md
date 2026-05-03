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

```bash
curl -s -X POST "http://127.0.0.1:8080/api/v1/agents/{AGENT_ID}/tasks/send" \
  -H "Content-Type: application/json" \
  -d '{
    "orderId": "{ORDER_ID}",
    "message": {
      "model": "hermes-agent",
      "input": "你好"
    }
  }'
```

## 5. 多轮对话

```bash
curl -s -X POST "http://127.0.0.1:8080/api/v1/agents/{AGENT_ID}/tasks/send" \
  -H "Content-Type: application/json" \
  -d "{\"orderId\":\"{ORDER_ID}\",\"message\":{\"model\":\"hermes-agent\",\"input\":\"第一轮\"}}"

curl -s -X POST "http://127.0.0.1:8080/api/v1/agents/{AGENT_ID}/tasks/send" \
  -H "Content-Type: application/json" \
  -d "{\"orderId\":\"{ORDER_ID}\",\"message\":{\"model\":\"hermes-agent\",\"input\":\"第二轮\"}}"
```

## 6. 收货

```bash
curl -s -X POST "http://127.0.0.1:8080/api/v1/orders/{ORDER_ID}/receipt"
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

