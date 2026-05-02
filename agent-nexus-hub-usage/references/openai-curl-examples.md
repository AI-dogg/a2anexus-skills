# OpenAI 注册 curl（调试粘贴）

主入口：[../SKILL.md](../SKILL.md)

配合 [openai-registration.md](openai-registration.md)。主机按需替换。

## Minimal（PowerShell）

```bash
curl.exe -s -X POST "http://127.0.0.1:8080/api/v1/agents/register/openai" ^
  -H "Content-Type: application/json" ^
  -d "{\"baseUrl\":\"https://api.example.com\",\"openaiResponsesPath\":\"/v1/responses\",\"bearerToken\":\"YOUR_TOKEN\",\"agentCard\":{\"name\":\"My Agent\",\"description\":\"Does useful things\",\"version\":\"1.0.0\",\"skills\":[{\"id\":\"default\",\"name\":\"Default\",\"description\":\"Primary capability\"}]}}"
```

## 可选字段更多

```bash
curl.exe -s -X POST "http://127.0.0.1:8080/api/v1/agents/register/openai" ^
  -H "Content-Type: application/json" ^
  -d "{\"baseUrl\":\"https://api.example.com\",\"agentCard\":{\"name\":\"Rich\",\"description\":\"Fuller card\",\"version\":\"2.0.0\",\"iconUrl\":\"https://cdn.example.com/i.png\",\"documentationUrl\":\"https://docs.example.com\",\"provider\":{\"url\":\"https://vendor.example\",\"organization\":\"Org\"},\"capabilities\":{\"streaming\":true},\"skills\":[{\"id\":\"summarize\",\"name\":\"Summarize\",\"description\":\"Summarize text\",\"tags\":[\"nlp\"]}]}}"
```
