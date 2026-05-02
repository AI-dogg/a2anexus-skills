# Agent Nexus（A2A Agent Nexus）

**A2A Agent Nexus** 是基于开放 **Agent2Agent（A2A）协议** 的基础设施型平台：在标准协议之上补齐 **安全、支付、推荐、调度**，让多 Agent 协作可发现、可计费、可审计；**不**新造协议替代 A2A，也 **不**把 Hub 自身当成 Catalog 里可被发现的下游 Agent。

## 一句话

调用语义仍是标准 A2A；默认路径是 **调用方 → Nexus 安全网关 → 下游 A2A Server**（含 SSE），由网关承担策略、配额、TLS 边界与连接级可观测；控制平面负责 **推荐、调度状态、支付与审计**，并与网关转发会话对齐。

## 要解决什么

A2A 已定义 Card、委托与通信，但生态仍缺统一的 **注册与发现（Catalog）**、 **价值交换与结算**、 **匹配与路由（推荐）**、以及与支付对齐的 **任务生命周期**。Agent Nexus 作为 **非与下游对等的控制平面**，把这些能力收拢到一处，而不是让调用方散落直连各地下游公网端点。

## 核心能力（四支柱）

| 支柱 | 要点 |
| --- | --- |
| **安全** | 注册与元数据校验、策略与审计；**网关为默认受控转发点** |
| **推荐** | Catalog、匹配与排序、费用与路由提示 |
| **调度** | 任务状态机、路由与重试、配额；与网关会话（如 `hubRequestId`）对齐 |
| **支付** | 链上执行轨以 **EIP-7702 + ERC-4337** 为主线叙事；产品默认 **Escrow**（锁定 → 经网关执行 → 完成证明对齐后 Settle/Release；争议与细则见文档） |

信任与排序当前主线为 **Hub 中心化评价**（评分、反作弊、争议与审计闭环），并与推荐、支付叙事衔接。

## 完整文档（Obsidian）

权威叙述、PRD、架构、网关、支付与 MVP 路线均在 **Obsidian Vault** 中维护：

- **入口索引（推荐从这里打开）**  
  `C:\Users\Administrator\Documents\Obsidian Vault\Projects\AgentNexus\00-导航.md`

- **主线正式文档目录**  
  `C:\Users\Administrator\Documents\Obsidian Vault\Projects\AgentNexus\正式文档\`  
  例如：`02-总览.md`、`03-核心流程.md`、`05-网关与转发.md`、`06-支付.md` 等。

- **Gateway 联调与 REST 细节**  
  `C:\Users\Administrator\Documents\Obsidian Vault\Projects\AgentNexus\开发文档\`

实现与接口以 **`a2anexus`** 代码仓库为准；叙述冲突时以 Vault 正式文档 + 代码事实核对。

---

*本文件夹中的 `SKILL.md` 与 `references/` 仅为 Cursor 集成时的执行速查，不构成平台定义；产品口径以上述 Obsidian 文档为准。*
