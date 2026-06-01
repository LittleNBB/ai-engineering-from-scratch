# 操作预算、迭代上限和成本治理器（Action Budgets, Iteration Caps, and Cost Governors）

> 一个中型电子商务智能体的月度 LLM 成本在团队启用"订单追踪"技能后从 $1,200 跳到 $4,800。这不是定价 bug。这是一个找到了新循环并在其中持续消费的智能体。微软的 Agent Governance Toolkit（2026 年 4 月 2 日）编纂了对此类别的防御：每请求 `max_tokens`、每任务 token 和美元预算、每日/月上限、迭代上限、分层模型路由、提示词缓存、上下文窗口化、昂贵动作的 HITL 检查点、预算违反时的紧急停止开关。Anthropic 的 Claude Code Agent SDK 以不同的名称发布了相同的原语。金融速度限制——例如 10 分钟内超过 $50 就切断访问——比月度上限更快地捕获循环。

**类型：** Learn（学习）
**语言：** Python（标准库，分层成本治理器模拟器）
**前置课程：** Phase 15 · 10（Permission modes），Phase 15 · 12（Durable execution）
**时间：** ~60 分钟

## 问题

自主智能体每轮都在花真实的钱。聊天机器人的坏输出是一条坏回复；智能体的坏循环是一张账单。行业有据可查的失败模式术语是"拒绝钱包攻击（Denial of Wallet）"——智能体持续推理、持续调用工具、持续计费，没有东西阻止它，因为没有东西被设计来阻止。

修复不是一个数字。它是不同时间尺度和粒度的限制栈：每请求、每任务、每小时、每天、每月。设计良好的栈在分钟内捕获失控循环，在小时内捕获慢泄漏，在一天内捕获坏发布。当智能体是长视界且自主时，相同的栈在任何时候都保持预算。

这是一课工程：数学是琐碎的，纪律才是团队失败的地方。下面的限制列表要么在微软 Agent Governance Toolkit 中命名，要么在 Anthropic Claude Code Agent SDK 文档中命名。

## 核心概念

### 成本治理器栈

1. **每请求 `max_tokens`。** 简单。防止任何单次调用发出无界的完成。
2. **每任务 token 预算。** 整个运行中不超过 N 个 token。硬停止在上限。
3. **每任务美元预算。** 与 token 相同但以货币计。Claude Code 中的 `max_budget_usd`。
4. **每工具调用上限。** 不超过 N 次 `WebFetch` 调用、N 次 `shell_exec` 调用等。
5. **迭代上限（`max_turns`）。** 总智能体循环迭代次数；防止无限推理循环。
6. **每分钟/每小时/每天/每月上限。** 滚动窗口。在不同时间尺度捕获泄漏。
7. **金融速度限制。** 例如"10 分钟内消费超过 $50 就切断访问"。在月度上限触发之前捕获基于循环的燃烧。
8. **分层模型路由。** 默认使用较小的模型；仅当分类器判断任务需要时才升级到较大的模型。
9. **提示词缓存。** 系统提示词和稳定上下文存储在提供商缓存中；重新发送的 token 成本接近零。
10. **上下文窗口化。** 压缩/摘要以保持活跃上下文低于阈值；直接降低 token 成本。
11. **昂贵动作的 HITL 检查点。** 在已知昂贵的动作（长时间工具调用、大下载、昂贵的模型升级）之前，需要人类点击。
12. **预算违反时的紧急停止开关。** 任何上限触发时会话中止。上限被记录；需要单独的重新启用路径。

### 为什么是栈，而不是一个上限

单月度上限在钱包空了之后才捕获失控智能体。单每请求上限在会话级别什么都捕获不到。不同的失败模式需要不同的时间尺度：

- **失控循环**（智能体卡在 5 秒重试中）：由速度限制捕获。
- **慢泄漏**（智能体每任务做约 2 倍预期工作）：由每日上限捕获。
- **坏发布**（新版本使用 5 倍 token）：由每周/每月上限捕获。
- **合法激增**（真实需求，不是 bug）：由带清晰日志的小时/天上限捕获。

### Claude Code 的预算表面

Claude Code Agent SDK 暴露（公开文档）：

- `max_turns` — 迭代上限。
- `max_budget_usd` — 美元上限；违反时会话中止。
- `allowed_tools` / `disallowed_tools` — 工具允许列表和拒绝列表。
- 工具使用前的钩子点，用于自定义成本核算。

与权限模式阶梯（Lesson 10）结合。没有 `max_budget_usd` 的 `autoMode` 会话是不受治理的自主性。Anthropic 明确将 Auto Mode 框架化为需要预算控制；分类器与成本正交。

### EU AI Act、OWASP Agentic Top 10

微软的 Agent Governance Toolkit 涵盖了 OWASP Agentic Top 10 和 EU AI Act 第 14 条（人类监督）要求。在欧盟的生产中，日志记录和上限执行不是可选的。

### 观察到的 $1,200 → $4,800 案例

微软文档中的真实案例：一个电子商务智能体在添加新工具后月成本翻了三倍。该工具允许智能体在每次会话中轮询订单状态。没有循环检测。没有每工具上限。没有周增长告警。修复是每工具上限加上日增长告警。这是一个模板：每个新工具表面都是一个新的潜在循环；每个新工具需要自己的上限和自己的告警。

## 动手实现

`code/main.py` 模拟一个有和没有分层成本治理器栈的智能体运行。模拟的智能体在几轮后漂移到轮询循环；分层栈在速度窗口内捕获它，而单月度上限要到几天后才触发。

## 实际应用

`outputs/skill-agent-budget-audit.md` 审计提议的智能体部署的成本治理器栈并标记缺失的层。

## 练习

1. 运行 `code/main.py`。确认在轮询循环轨迹上速度限制在迭代上限之前触发。现在禁用速度限制并测量在迭代上限捕获之前智能体"花费"了多少。

2. 为浏览器智能体（Lesson 11）设计一套每工具上限。哪个工具需要最严格的上限？哪个工具可以无限制运行而没有风险？

3. 阅读微软 Agent Governance Toolkit 文档。列出工具包命名的每种上限类型。将每种映射到一个失败模式（失控循环、慢泄漏、坏发布、激增）。

4. 为一个真实任务（例如"分类仓库中的 50 个 issue"）定价一个隔夜无人值守运行。将 `max_budget_usd` 设为你点估计的 2 倍。论证 2 倍的理由。

5. Claude Code 的 `max_budget_usd` 基于会话聚合成本触发。设计一个你会在外部强制执行的补充速度限制。什么触发切断，重新启用是什么样的？

## 关键术语

| 术语 | 常见说法 | 实际含义 |
|---|---|---|
| Denial of Wallet（拒绝钱包攻击） | "失控账单" | 智能体循环产生没有上限阻止的消费 |
| max_tokens | "每请求上限" | 单次完成大小的天花板 |
| max_turns | "迭代上限" | 会话中智能体循环迭代次数的天花板 |
| max_budget_usd | "美元紧急停止开关" | 会话成本上限；违反时中止 |
| Velocity limit（速度限制） | "速率上限" | 短窗口内的消费限制（如 $50 / 10 分钟） |
| Tiered routing（分层路由） | "小模型优先" | 便宜模型默认；仅当分类器保证时才升级 |
| Prompt caching（提示词缓存） | "缓存的系统提示词" | 提供商端缓存将重新发送 token 成本降至接近零 |
| HITL checkpoint（HITL 检查点） | "人类审批门控" | 昂贵动作前需要人类点击 |

## 延伸阅读

- [Anthropic Claude Code Agent SDK — agent loop and budgets](https://code.claude.com/docs/en/agent-sdk/agent-loop) — `max_turns`、`max_budget_usd`、工具允许列表
- [Microsoft Agent Framework — human-in-the-loop and governance](https://learn.microsoft.com/en-us/agent-framework/workflows/human-in-the-loop) — 成本治理器检查点
- [Anthropic — Claude Managed Agents overview](https://platform.claude.com/docs/en/managed-agents/overview) — 提供商端成本控制
- [Anthropic — Prompt caching (Claude API docs)](https://platform.claude.com/docs/en/prompt-caching) — 缓存机制
- [Anthropic — Measuring agent autonomy in practice](https://www.anthropic.com/research/measuring-agent-autonomy) — 长视界智能体的成本概况