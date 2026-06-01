# 群聊与发言选择（Group Chat and Speaker Selection）

> AutoGen GroupChat 和 AG2 GroupChat 在 N 个智能体之间共享一个对话；选择器函数（LLM、轮询或自定义）选择谁下一个发言。这是涌现式多智能体对话的原型——智能体不知道自己在静态图中的角色，它们只是对共享池做出反应。AutoGen v0.2 的 GroupChat 语义在 AG2 分支中得以保留；AutoGen v0.4 将其重写为事件驱动的 Actor 模型。微软于 2026 年 2 月将 AutoGen 转入维护模式，并将其与 Semantic Kernel 合并为 Microsoft Agent Framework（2026 年 2 月 RC）。GroupChat 原语在 AG2 和 Microsoft Agent Framework 中都存活——学一次，到处用。

**类型：** Learn + Build（学习 + 构建）
**语言：** Python（标准库）
**前置课程：** Phase 16 · 04（Primitive Model）
**时间：** ~60 分钟

## 问题

静态图（LangGraph）在工作流已知时表现出色。真实对话不是静态的：有时编码者问审阅者，有时问研究者，有时问写作者。硬编码每个可能的移交会导致边爆炸。你想要*智能体对共享池做出反应*，由某个函数决定谁下一个说话。

这正是 AutoGen GroupChat 所做的。

## 核心概念

### 形态

```
              ┌─── 共享池 ────┐
              │  m1  m2  m3  ... │
              └────────┬────────┘
                       │（所有人读所有消息）
      ┌───────┬────────┼────────┬───────┐
      ▼       ▼        ▼        ▼       ▼
    智能体 A  智能体 B  智能体 C  智能体 D  选择器
                                           │
                                           ▼
                                  "下一个发言者 = C"
```

每个智能体看到每条消息。每轮调用一个选择器函数来选择谁下一个发言。

### 三种选择器风格

**轮询（Round-robin）。** 固定循环。确定性。在 N 个智能体上线性扩展，但忽略上下文——即使话题是法律审查，编码者也会轮到。

**LLM 选择。** 调用一个 LLM 读取最近的池并返回最佳的下一个发言者。上下文感知但慢：每轮增加一次 LLM 调用。AutoGen 的默认方式。

**自定义。** 一个带有你想要的任何逻辑的 Python 函数。典型做法：LLM 选择加上回退规则（例如，"编码者之后总是给验证者轮次"）。

### ConversableAgent API

```python
agent = ConversableAgent(
    name="coder",
    system_message="You write Python.",
    llm_config={...},
)
chat = GroupChat(agents=[coder, reviewer, tester], messages=[])
manager = GroupChatManager(groupchat=chat, llm_config={...})
```

`GroupChatManager` 持有选择器。当一个智能体完成一轮时，管理器调用选择器，选择器返回下一个智能体。循环持续直到终止条件。

### 终止

三种常见模式：

- **最大轮数。** 对总轮次的硬性上限。
- **"TERMINATE" 标记。** 智能体可以发出哨兵消息；当它出现时管理器停止。
- **目标达成检查。** 每轮运行一个轻量级验证器，完成时停止聊天。

### AutoGen → AG2 分裂与 Microsoft Agent Framework 合并

2025 年初，微软开始围绕事件驱动 Actor 模型对 AutoGen（v0.4）进行重大重写。社区将 AutoGen v0.2 的 GroupChat 语义分叉为 AG2，保留了早期采用者已集成的 API。

2026 年 2 月，微软宣布 AutoGen 转入维护模式，事件驱动 Actor 模型并入 **Microsoft Agent Framework**（2026 年 2 月 RC，现已与 Semantic Kernel 合并）。GroupChat 概念在两条路线上都存活；实现细节不同。AG2 是 v0.2 兼容代码的首选上游。

### GroupChat 适用场景

- **涌现式对话。** 你不想预先连接每个可能的下一个发言者。
- **角色混合任务。** 编码者问研究者，研究者问档案员，档案员问编码者。流不是 DAG。
- **探索性问题解决。** 想象"头脑风暴会议"，而非"装配线"。

### 失败场景

- **严格确定性。** LLM 选择器可能不一致。相同提示词，不同运行，不同下一个发言者。
- **谄媚级联。** 智能体顺从发言最自信的那位。用显式对抗性提示词来对抗。
- **上下文膨胀。** 每个智能体读取每条消息；10 轮后上下文巨大。使用投影（Lesson 15）来限定视图。
- **热门发言者。** 一个智能体主导对话，因为选择器偏好其专长。将发言平衡作为选择器特性引入。

### 群聊 vs 监督者

相同的原语，不同的默认值：

- 监督者：一个智能体规划，其他执行。选择器是"问规划者该做什么"。
- 群聊：所有智能体是对等的；选择器是对共享池的函数。

两者都使用 Lesson 04 的四个原语。群聊默认使用 LLM 选择的编排和全池共享状态。

## 动手实现

`code/main.py` 用标准库从头实现了一个 GroupChat。三个智能体（编码者、审阅者、管理者），轮询和 LLM 选择变体，以及基于 `TERMINATE` 标记的终止。

演示打印对话记录加上两种变体的选择器决策跟踪。

运行：

```
python3 code/main.py
```

## 实际应用

`outputs/skill-groupchat-selector.md` 为给定任务配置 GroupChat 选择器——轮询 vs LLM 选择 vs 自定义，以及使用什么选择器输入（最近消息、智能体专长、轮次计数）。

## 交付建议

检查清单：

- **最大轮数上限。** 必须有。典型任务 10-20。
- **发言平衡指标。** 跟踪每个智能体的轮次；当不平衡超过阈值时告警。
- **终止标记。** `TERMINATE` 或专用的验证者智能体。
- **投影或范围记忆。** 约 10 条消息后，考虑只给每个智能体范围视图以防止上下文膨胀。
- **选择器日志。** 对于 LLM 选择变体，记录选择器的输入和选择。否则调试不可能。

## 练习

1. 运行 `code/main.py`。对比轮询 vs LLM 选择下的对话。哪种情况下哪个智能体主导？
2. 在选择器中添加"每个智能体最多发言次数"规则。它如何影响记录？
3. 实现目标达成终止：当审阅者返回"approved"时停止。它在轮数上限之前触发的频率如何？
4. 阅读 AutoGen 稳定版文档中的 GroupChat 部分（https://microsoft.github.io/autogen/stable/user-guide/core-user-guide/design-patterns/group-chat.html）。识别 `GroupChatManager` 使用的默认选择器。
5. 阅读 AG2 仓库（https://github.com/ag2ai/ag2）并比较其 v0.2 GroupChat 与 v0.4 事件驱动版本。v0.4 添加了什么具体属性（吞吐量、容错性、可组合性）？

## 关键术语

| 术语 | 常见说法 | 实际含义 |
|------|---------|---------|
| GroupChat（群聊） | "智能体在一个聊天室" | 共享消息池 + 选择器函数。AutoGen / AG2 原语。 |
| Speaker selection（发言选择） | "谁下一个说话" | 选择下一个智能体的函数。轮询、LLM 选择或自定义。 |
| GroupChatManager（群聊管理器） | "会议主持人" | 拥有选择器并循环轮次的 AutoGen 组件。 |
| ConversableAgent（可对话智能体） | "基础智能体" | AutoGen 基类；可以发送和接收消息的智能体。 |
| Termination token（终止标记） | "停止词" | 结束聊天的哨兵字符串（通常是 `TERMINATE`）。 |
| Hot speaker（热门发言者） | "一个智能体主导" | 选择器持续选择同一个智能体的失败模式。 |
| Context bloat（上下文膨胀） | "池无限增长" | 每个智能体读取每条先前消息；上下文随轮次增长。 |
| Projection（投影） | "范围视图" | 共享池的角色特定视图，防止上下文膨胀。 |

## 延伸阅读

- [AutoGen group chat docs](https://microsoft.github.io/autogen/stable/user-guide/core-user-guide/design-patterns/group-chat.html) — 参考实现
- [AG2 repo](https://github.com/ag2ai/ag2) — 社区 AutoGen v0.2 延续
- [Microsoft Agent Framework docs](https://microsoft.github.io/agent-framework/) — 合并后的继任者，2026 年 2 月 RC
- [AutoGen v0.4 release notes](https://microsoft.github.io/autogen/stable/) — 事件驱动 Actor 模型重写细节