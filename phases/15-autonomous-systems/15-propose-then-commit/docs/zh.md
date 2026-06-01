# 人在回路：先提案再提交（Human-in-the-Loop: Propose-Then-Commit）

> 2026 年对 HITL 的共识是具体的。不是"智能体问，用户点批准"。而是先提案再提交：提议的动作被持久化到带幂等键的持久存储中；以意图、数据溯源、触及的权限、爆炸半径和回滚计划呈现给审阅者；仅在正向确认后提交；执行后验证以确认副作用确实发生了。LangGraph 的 `interrupt()` 加 PostgreSQL 检查点、Microsoft Agent Framework 的 `RequestInfoEvent` 和 Cloudflare 的 `waitForApproval()` 都实现了相同的形状。经典的失败模式是橡皮图章审批："批准？"在没有审查的情况下被点击。有据可查的缓解措施是带显式检查清单的挑战-响应。

**类型：** Learn（学习）
**语言：** Python（标准库，带幂等性的先提案再提交状态机）
**前置课程：** Phase 15 · 12（Durable execution），Phase 15 · 14（Tripwires）
**时间：** ~60 分钟

## 问题

智能体执行一个动作。用户必须决定：批准还是不批准。如果决定是即时的，它可能不是审查。如果决定是结构化的，它慢但可信。工程问题是如何使结构化审查成为阻力最小的路径。

2023 年代的 HITL 模式是同步提示："智能体想发送邮件给 X，内容为 Y——批准？"用户点击批准。每个人都觉得系统安全。在实践中，这个表面被大量橡皮图章：用户快速批准，批准几乎不预测什么，当智能体出错时，审计跟踪显示用户无法回忆的长批准历史。

2026 年的模式——先提案再提交——将 HITL 移到持久化基底上，附加结构化元数据，并要求正向提交。每个托管智能体 SDK 都发布了版本：LangGraph `interrupt()`、Microsoft Agent Framework `RequestInfoEvent`、Cloudflare `waitForApproval()`。API 名称不同；形状相同。

## 核心概念

### 先提案再提交状态机

1. **提案（Propose）。** 智能体产生一个提议的动作。持久化到存储（PostgreSQL、Redis、Durable Object）。包括：
   - 意图（智能体为什么做这个）
   - 数据溯源（什么来源导致了这个提案）
   - 触及的权限（哪些范围/文件/端点）
   - 爆炸半径（最坏情况是什么）
   - 回滚计划（如果提交了，如何撤销）
   - 幂等键（每个提案唯一；重新提交返回同一条记录）
2. **呈现（Surface）。** 审阅者看到带所有元数据的提案。审阅者是人（不是智能体审阅自己）。
3. **提交（Commit）。** 正向确认。动作执行。
4. **验证（Verify）。** 执行后，副作用被回读并确认。如果验证步骤失败，系统处于已知坏状态并触发告警。

### 幂等键

没有幂等键，瞬态故障后的重试可能双重执行已批准的动作。具体示例：用户批准"从 A 转 $100 到 B"。网络闪断。工作流重试。用户批准了一次但转账执行了两次。幂等键将批准绑定到单个、唯一的副作用；第二次执行是空操作。

这与 Stripe 和 AWS API 使用的幂等模式相同。微软 Agent Framework 文档中明确将此复用于智能体批准。

### 持久化：为什么批准比进程活得更久

批准等待室是智能体不拥有的状态。工作流被暂停（Lesson 12）。当批准到达时，工作流从该精确点恢复。这就是为什么 LangGraph 将 `interrupt()` 与 PostgreSQL 检查点配对而非仅内存状态——两天后的批准仍然能找到完整的工作流。

### 橡皮图章审批和挑战-响应缓解

HITL 的默认 UI（"批准"/"拒绝"按钮）产生没有真正审查的快速批准。有据可查的缓解措施：挑战-响应检查清单，在启用批准按钮之前要求对特定问题的正向回答。具体形状：

- "你理解这触及什么资源吗？ [ ]"
- "你验证了爆炸半径可接受吗？ [ ]"
- "如果失败你有回滚计划吗？ [ ]"

不是为了官僚而官僚——而是一个强制函数。不能勾选框的审阅者要么请求澄清（升级）要么拒绝（安全默认）。Anthropic 智能体安全研究明确引用检查清单驱动的 HITL 作为橡皮图章审批模式的缓解措施。

### 什么算作重要动作

不是每个动作都需要先提案再提交。2026 年指导：

- **重要动作**（总是 HITL）：不可逆写入、金融交易、出站通信、生产数据库变更、破坏性文件系统操作。
- **可逆动作**（有时 HITL）：本地文件编辑、暂存环境变更、有清晰回滚的可逆写入。
- **读取和检查**（永不 HITL）：读取文件、列出资源、调用只读 API。

### 动作后验证

"提交已运行"不等于"副作用已发生"。网络分区和竞态条件可能产生认为成功而后端未持久化的工作流。验证步骤在提交后重新读取目标资源以确认。这与带 `RETURNING` 子句的数据库事务或 AWS `PutObject` 后 `GetObject` 的模式相同。

### EU AI Act 第 14 条

第 14 条要求欧盟高风险 AI 系统的有效人类监督。"有效"不是装饰性的。监管语言明确排除橡皮图章模式。带挑战-响应的先提案再提交是在微软 Agent Governance Toolkit 合规文档中通过第 14 条审查的形状。

## 动手实现

`code/main.py` 用标准库 Python 实现了一个先提案再提交状态机。持久存储是 JSON 文件。幂等键是 (thread_id, action_signature) 的哈希。驱动器模拟三种情况：干净的批准流程、瞬态故障后的重试（不能双重执行）和橡皮图章默认 vs 挑战-响应流程。

## 实际应用

`outputs/skill-hitl-design.md` 审查提议的 HITL 工作流的先提案再提交形状，并标记缺失的元数据、幂等性、验证或挑战-响应层。

## 练习

1. 运行 `code/main.py`。确认对已批准提案的重试使用持久化记录且不重新执行。现在将幂等键改为包含时间戳并展示重试双重执行。

2. 用 `rollback` 字段扩展提案记录。模拟一个验证步骤失败的执行。展示回滚自动触发。

3. 阅读微软 Agent Framework 的 `RequestInfoEvent` 文档。识别 API 包含但玩具引擎缺失的一个元数据字段。添加它并解释它保护免受什么。

4. 为特定动作设计一个挑战-响应检查清单（例如"发布到公共 Twitter 账户"）。审阅者必须回答哪三个问题？为什么是这三个？

5. 选择一个同步"批准？"提示就足够的情况（不需要持久存储）。解释为什么，并命名你接受的风险类别。

## 关键术语

| 术语 | 常见说法 | 实际含义 |
|---|---|---|
| Propose-then-commit（先提案再提交） | "两阶段审批" | 持久化提案 + 正向提交 + 验证 |
| Idempotency key（幂等键） | "重试安全 token" | 每个提案唯一；第二次执行为空操作 |
| Data lineage（数据溯源） | "它从哪来" | 导致提案的具体来源内容 |
| Blast radius（爆炸半径） | "最坏情况" | 动作出错时的影响范围 |
| Rubber-stamp（橡皮图章） | "快速审批" | 没有真正审查就点击"批准" |
| Challenge-and-response（挑战-响应） | "强制检查清单" | 审阅者必须正向确认特定问题 |
| RequestInfoEvent | "MS 智能体框架原语" | 带结构化元数据的持久化 HITL 请求 |
| `interrupt()` / `waitForApproval()` | "框架原语" | LangGraph / Cloudflare 的等价物 |

## 延伸阅读

- [Microsoft Agent Framework — Human in the loop](https://learn.microsoft.com/en-us/agent-framework/workflows/human-in-the-loop) — `RequestInfoEvent`、持久化批准
- [Cloudflare Agents — Human in the loop](https://developers.cloudflare.com/agents/concepts/human-in-the-loop/) — `waitForApproval()` 和 Durable Objects
- [Anthropic — Measuring agent autonomy in practice](https://www.anthropic.com/research/measuring-agent-autonomy) — HITL 作为长视界风险的缓解措施
- [EU AI Act — Article 14: Human oversight](https://artificialintelligenceact.eu/article/14/) — 高风险系统的监管基线
- [Anthropic — Claude's Constitution (January 2026)](https://www.anthropic.com/news/claudes-constitution) — 围绕监督的宪法框架