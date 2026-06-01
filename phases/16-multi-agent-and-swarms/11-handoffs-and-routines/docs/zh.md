# 移交与例程 —— 无状态编排（Handoffs and Routines — Stateless Orchestration）

> OpenAI 的 Swarm（2024 年 10 月）将多智能体编排提炼为两个原语：**例程（routines）**（指令 + 工具作为系统提示词）和**移交（handoffs）**（一个返回另一个 Agent 的工具）。没有状态机，没有分支 DSL——LLM 通过调用正确的移交工具来路由。OpenAI Agents SDK（2025 年 3 月）是其生产继任者。Swarm 本身仍是最简洁的概念参考——其全部源码只有几百行。这个模式之所以流行，是因为 API 表面大约就是"agent = prompt + tools; handoff = 返回 agent 的函数"。局限：无状态，所以记忆是调用者的问题。

**类型：** Learn + Build（学习 + 构建）
**语言：** Python（标准库）
**前置课程：** Phase 16 · 04（Primitive Model）
**时间：** ~60 分钟

## 问题

每个多智能体框架都想让你学习它的 DSL：LangGraph 的节点和边、CrewAI 的团队和任务、AutoGen 的 GroupChat 和管理器。DSL 是真正的抽象，但它们让事情感觉比必要的更重。

Swarm 朝相反方向推进：使用模型已经具备的工具调用能力。移交变成工具调用。编排器是当前持有对话的智能体。状态机隐含在智能体的系统提示词中。

## 核心概念

### 两个原语

**Routine（例程）。** 定义智能体角色和可用工具的系统提示词。把它想象成一个作用域化的指令集："你是一个分诊智能体；如果用户问退款，移交给退款智能体。"

**Handoff（移交）。** 智能体可以调用的一个工具，返回一个新的 Agent 对象。Swarm 运行时检测到 Agent 返回值并在下一轮切换活跃智能体。

这就是全部抽象。

```python
def transfer_to_refunds():
    return refund_agent  # Swarm 看到 Agent 返回 → 切换活跃智能体

triage_agent = Agent(
    name="triage",
    instructions="Route the user to the right specialist.",
    functions=[transfer_to_refunds, transfer_to_sales, transfer_to_support],
)
```

分诊智能体的系统提示词让它根据用户消息选择正确的移交。LLM 的工具调用完成路由。

### 为什么它流行

- **小 API。** 只需学两个概念。
- **使用模型已经会的。** 工具调用在各提供商中已经是生产级的。
- **没有状态机负担。** 你不需要描述图；智能体的提示词描述它们移交给谁。

### 无状态的权衡

Swarm 在运行之间是显式无状态的。框架在运行期间保持消息历史，但不持久化任何东西。记忆、连续性、长时间运行的任务——都是调用者的问题。

在生产中（OpenAI Agents SDK，2025 年 3 月），这是主要改变之一：SDK 在保持移交原语的同时添加了内置的会话管理、防护和追踪。

### Swarm/移交适用场景

- **分诊模式。** 前线智能体将用户路由到专家。
- **基于技能的移交。** "如果任务需要代码，调用编码者；如果需要研究，调用研究者。"
- **短而有界的对话。** 客户支持、FAQ 到工单、简单工作流。

### Swarm 困难场景

- **需要共享记忆的长会话。** 移交将对话状态重置为新智能体的提示词加历史。没有调用者管理的记忆就没有跨智能体的持久状态。
- **并行执行。** 移交一次一个——活跃智能体切换。并行需要调用者编排多个 Swarm 运行。
- **审计和重放。** 无状态运行难以精确重放；LLM 的移交选择不是确定性的。

### OpenAI Agents SDK（2025 年 3 月）

生产继任者添加了：

- **会话状态。** 跨运行的持久线程。
- **防护。** 输入/输出验证钩子。
- **追踪。** 每次工具调用和移交都被记录。
- **移交过滤器。** 控制移交时转移什么上下文。

移交原语存活；生产人体工学围绕它被添加。

### Swarm vs GroupChat

两者都使用 LLM 驱动的路由，但它们在**谁选择下一个**上不同：

- GroupChat：一个选择器（函数或 LLM）从外部选择下一个发言者。
- Swarm：当前智能体通过调用移交工具选择其继任者。

Swarm 是"智能体决定下一步"；GroupChat 是"管理者决定下一步"。Swarm 的决策位于活跃智能体的工具调用中；GroupChat 的位于 `GroupChatManager` 中。

## 动手实现

`code/main.py` 从头实现 Swarm：一个 Agent 数据类、一个移交机制（工具返回 Agent）和一个检测智能体切换的运行循环。

演示：一个分诊智能体路由到退款、销售或支持专家。每个专家有自己的工具。运行循环打印每次移交。

运行：

```
python3 code/main.py
```

## 实际应用

`outputs/skill-handoff-designer.md` 为给定任务设计移交拓扑：存在哪些智能体、它们可以调用哪些移交、什么上下文被转移。

## 交付建议

检查清单：

- **移交日志。** 每次移交写入一个跟踪事件，包含来源智能体、目标智能体、上下文快照。
- **上下文转移规则。** 决定移交时移动什么：完整历史（昂贵）、最近 N 条消息或摘要。
- **移交时的防护。** 到具有不同工具权限的专家的移交必须经过认证——否则提示词注入可以强制进行非预期的移交。
- **循环检测。** 两个智能体来回移交是常见的失败；用简单的最近 K 次环形检查来检测。
- **回退智能体。** 如果移交目标不存在，回退到安全默认值。

## 练习

1. 运行 `code/main.py`，分诊到退款智能体。确认第二轮的活跃智能体是退款。
2. 添加循环检测规则：如果相同的两个智能体连续移交 3 次，强制退出。设计回退方案。
3. 阅读 OpenAI Agents SDK 文档中的移交过滤器部分。实现一个"移交时摘要"版本：传出的智能体在传入的智能体接管之前将上下文压缩为要点摘要。
4. 对比 Swarm 移交与 GroupChatManager 选择器。哪种模式使提示词注入更严重，为什么？
5. 阅读 Swarm cookbook（https://developers.openai.com/cookbook/examples/orchestrating_agents）。识别 Swarm 做出的一个明确设计决策，该决策被 OpenAI Agents SDK 改变或保留。

## 关键术语

| 术语 | 常见说法 | 实际含义 |
|------|---------|---------|
| Routine（例程） | "智能体提示词" | 系统提示词 + 工具列表。定义角色和可用移交。 |
| Handoff（移交） | "转移到另一个智能体" | 活跃智能体可以调用的一个工具，返回新的 Agent。运行时切换活跃智能体。 |
| Stateless（无状态） | "运行之间没有记忆" | Swarm 不持久化任何东西；记忆是调用者的责任。 |
| Active agent（活跃智能体） | "现在谁在说话" | 当前持有对话的智能体。移交改变这一点。 |
| Context transfer（上下文转移） | "移交时移动什么" | 传入智能体看到什么历史的策略：完整、最近 N 条或摘要。 |
| Handoff loop（移交循环） | "智能体乒乓" | 两个智能体持续互相移交的失败模式。 |
| OpenAI Agents SDK | "生产版 Swarm" | 2025 年 3 月继任者；在移交原语之上添加会话、防护、追踪。 |
| Handoff filter（移交过滤器） | "转移时的门控" | SDK 特性，在移交边界检查和修改上下文。 |

## 延伸阅读

- [OpenAI cookbook — Orchestrating Agents: Routines and Handoffs](https://developers.openai.com/cookbook/examples/orchestrating_agents) — 参考阐述
- [OpenAI Swarm repo](https://github.com/openai/swarm) — 原始实现，保留为概念参考
- [OpenAI Agents SDK docs](https://openai.github.io/openai-agents-python/) — 带会话和追踪的生产继任者
- [Anthropic handoff-in-Claude notes](https://docs.anthropic.com/en/docs/claude-code) — Claude Code 子智能体如何通过 `Task` 使用类似移交的模式