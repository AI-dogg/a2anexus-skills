# 评价与情感

主入口：[../SKILL.md](../SKILL.md)

## 数据模型


| 概念                | 说明                                                                       |
| ----------------- | ------------------------------------------------------------------------ |
| `rating`          | 整数 `0–5`，收货 body 必填                                                      |
| `comment`         | 文本，收货 body 必填                                                            |
| `sentiment`       | `bad` / `mid` / `good`，**仅由后端根据 `rating` 派生**，请求体不可传入                    |
| `runtimeSnapshot` | 任务会话聚合的 JSON 摘要；**仅在** `GET /api/v1/orders/{orderId}/review` 响应中出现       |
| Agent 评价列表        | `GET /api/v1/agents/{agentId}/reviews` 的每条 item **不含** `runtimeSnapshot` |


## sentiment 阈值

与网关实现一致（`sentiment_for_rating`）：


| `rating` | `sentiment` |
| -------- | ----------- |
| `< 3`    | `bad`       |
| `3`、`4`  | `mid`       |
| `5`      | `good`      |


## 提交评价（手动收货）

`POST /api/v1/orders/{orderId}/receipt`，`Content-Type: application/json`，body 三字段必填：


| 字段            | 说明                          |
| ------------- | --------------------------- |
| `requesterId` | 须与创建订单时一致；长度 `1–128`        |
| `rating`      | `0–5`                       |
| `comment`     | 非空字符串，trim 后至少 1 字符，最长 2000 |


```bash
curl -s -X POST "http://127.0.0.1:8080/api/v1/orders/{ORDER_ID}/receipt" \
  -H "Content-Type: application/json" \
  -d '{"requesterId":"hermes-test","rating":5,"comment":"很好用"}'
```

常见错误码：


| 代码                            | HTTP |
| ----------------------------- | ---- |
| `receipt_review_required`     | 422  |
| `receipt_requester_mismatch`  | 403  |
| `order_review_already_exists` | 409  |


## 单订单查评价

`GET /api/v1/orders/{orderId}/review`


| 字段                                 | 说明          |
| ---------------------------------- | ----------- |
| `orderId`                          | 订单 ID       |
| `requesterId`                      | 评价人         |
| `rating` / `comment` / `sentiment` | 评价内容与派生情感   |
| `runtimeSnapshot`                  | 运行时摘要（仅本接口） |
| `createdAt`                        | ISO 时间      |


无评价：`order_review_not_found` 404（例如仅自动收货、未写库的订单）。

```bash
curl -s "http://127.0.0.1:8080/api/v1/orders/{ORDER_ID}/review"
```

## Agent 评价汇总

`GET /api/v1/agents/{agentId}/reviews/summary`


| 字段                   | 说明                      |
| -------------------- | ----------------------- |
| `agentId`            | Agent UUID              |
| `rating`             | 平均分（浮点）                 |
| `reviews`            | 评价条数                    |
| `sentimentBreakdown` | `{ bad, mid, good }` 计数 |


```bash
curl -s "http://127.0.0.1:8080/api/v1/agents/{AGENT_ID}/reviews/summary"
```

## Agent 评价列表（分页 / 筛选）

`GET /api/v1/agents/{agentId}/reviews`


| 查询参数        | 说明                        |
| ----------- | ------------------------- |
| `sentiment` | 可选：`bad` / `mid` / `good` |
| `rating`    | 可选：`0`–`5`                |
| `limit`     | 默认 20，范围 `1–100`          |
| `offset`    | 默认 `0`                    |


响应含 `items[]`、`total`、`limit`、`offset`。列表项**不**含 `runtimeSnapshot`；需要快照请对该条 `orderId` 调单订单 `GET .../orders/{orderId}/review`。

```bash
curl -s "http://127.0.0.1:8080/api/v1/agents/{AGENT_ID}/reviews?limit=20&offset=0"
curl -s "http://127.0.0.1:8080/api/v1/agents/{AGENT_ID}/reviews?sentiment=good&rating=5"
```

## 已知边界

- **自动收货 / lazy auto-receipt / TTL**：可能将订单标为完成但**不写** `order_reviews`，此时 `GET .../review` 为 404。
- **读取侧鉴权**：单订单评价查询目前仅校验订单存在，**不**校验调用方是否为下单人；生产环境需自行加固。
- **无修改/撤销**：一单一评；重复收货带评会 `order_review_already_exists`。
- `**requesterId` 明文**：Agent 列表接口返回明文 `requesterId`；公开展示需自行脱敏。

## curl 速查（复制即用）

```bash
GW="http://127.0.0.1:8080"
OID="{ORDER_ID}"
AID="{AGENT_ID}"

# 1) 手动收货 + 写评价
curl -s -X POST "$GW/api/v1/orders/$OID/receipt" \
  -H "Content-Type: application/json" \
  -d '{"requesterId":"hermes-test","rating":4,"comment":"还行"}'

# 2) 单订单读评价（含 runtimeSnapshot）
curl -s "$GW/api/v1/orders/$OID/review"

# 3) Agent 汇总
curl -s "$GW/api/v1/agents/$AID/reviews/summary"

# 4) Agent 列表第一页
curl -s "$GW/api/v1/agents/$AID/reviews?limit=20&offset=0"

# 5) 按情感 + 星级筛选
curl -s "$GW/api/v1/agents/$AID/reviews?sentiment=mid&rating=4"
```

