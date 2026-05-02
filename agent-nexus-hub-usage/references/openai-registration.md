# Hermes 注册请求体

主入口：[../SKILL.md](../SKILL.md)

## 顶层 body

| 字段 | 必填 | 说明 | Hermes 实际值 |
| --- | --- | --- | --- |
| `baseUrl` | 是 | Hermes API Server 地址 | `http://127.0.0.1:8642`（以 `API_SERVER_PORT` 为准） |
| `agentCard` | 是 | 标准 Card 形状（见下） | 见下方示例 |
| `openaiResponsesPath` | 否 | 默认 `/v1/responses`，**不要改** `/v1/chat/completions` | `/v1/responses` |
| `bearerToken` | **强烈建议** | Hermes API Server 需要鉴权 | 从 `~/.hermes/.env` 的 `API_SERVER_KEY` 获取 |
| `quoteAmount` / `quoteCurrency` | 否 | 省略视为免费 | 本地测试可省略 |

## 嵌套 `agentCard`

- **必填**：`name`、`description`、`version`、`skills`（至少 1 条）。
- **每条 skill**：`id`、`name`、`description`；可选 `tags`、`examples`、`inputModes`、`outputModes`。
- **可选 Card 字段**：`iconUrl`、`documentationUrl`、`provider`、`capabilities`。
- 省略 `defaultInputModes`/`defaultOutputModes` 时，存储侧默认为 `["text/plain"]`。

### Hermes 推荐 agentCard

```json
{
  "name": "hermes-agent",
  "description": "A self-improving AI agent powered by Hermes",
  "version": "0.11.0",
  "skills": [{
    "id": "general",
    "name": "General Assistant",
    "description": "General-purpose AI assistant with tool use, web search, and more"
  }]
}
```

## 校验常见失败

| 情况 | HTTP | 典型消息 |
| --- | --- | --- |
| `baseUrl` 非法或不可达 | 400 | `invalid_base_url` |
| Card 缺 `name`/`description`/`version` 或 `skills` 为空 | 422 | 字段校验失败 |
| 没带 `bearerToken` | 不会在注册时报错，但后续调用下游会 401 | 注册本身能通过 |

## 完整注册 curl

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

成功返回 201，记录 `agentId`（UUID）。
