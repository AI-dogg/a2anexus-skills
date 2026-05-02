# OpenAI 下游注册请求体

主入口：[../SKILL.md](../SKILL.md)

## 与 MVP Skill 的关系

MVP 主干只需记住：**`POST /api/v1/agents/register/openai`** 必填 **`baseUrl`** + 嵌套 **`agentCard`**；成功 **`201`** 后持久化 **`agentId`**。下列表格用于 **核对字段、排查 422/400**，不必一次性全部读入上下文。

仅在调用 **`POST /api/v1/agents/register/openai`** 且需要完整字段说明时阅读。

网关会 **注入** `supportedInterfaces`（如 `OPENAI_RESPONSES` 绑定）；请求里 **不要** 提交 `supportedInterfaces`、`signatures`。

## 顶层 body

| 字段 | 必填 | 说明 |
| --- | --- | --- |
| `baseUrl` | 是 | `http`/`https`；服务端会规范化 |
| `agentCard` | 是 | 标准 Card 形状（见下） |
| `openaiResponsesPath` | 否 | 默认 `/v1/responses` |
| `bearerToken` | 否 | 网关转发下游时使用 |
| `quoteAmount` / `quoteCurrency` | 否 | 省略视为免费（对外 quote 常为 `"0"`） |

## 嵌套 `agentCard`（摘要）

- **必填**：`name`、`description`、`version`、`skills`（至少 1 条）。
- **每条 skill**：`id`、`name`、`description`；可选 `tags`、`examples`、`inputModes`、`outputModes` 等。
- **可选 Card 字段**：`iconUrl`、`documentationUrl`、`provider`、`capabilities` 等。
- **省略 `defaultInputModes`/`defaultOutputModes`** 时，存储侧可为 `["text/plain"]`。

## 校验常见失败

| 情况 | 典型结果 |
| --- | --- |
| `baseUrl` 非法 | `400` `invalid_base_url` |
| Card 缺字段 / `skills` 为空 | `422` |

## curl 示例

见同目录 [openai-curl-examples.md](openai-curl-examples.md)。
