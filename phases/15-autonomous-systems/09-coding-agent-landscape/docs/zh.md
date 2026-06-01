# 自主编码智能体版图（2026）（The Autonomous Coding Agent Landscape (2026)）

> SWE-bench Verified 在不到三年内从 4% 到 80.9%。同一个 Claude Sonnet 4.5 在 SWE-agent v1 上得分 43.2%，在 Cline autonomous 上得分 59.8%——模型周围的脚手架现在和模型本身一样重要。OpenHands（前 OpenDevin）是最活跃的 MIT 许可平台，其 CodeAct 循环直接在沙箱中执行 Python 动作，而非 JSON 工具调用。头条数字隐藏了一个方法论问题：500 个 SWE-bench Verified 任务中有 161 个只需要 1-2 行更改，而 SWE-bench Pro（10+ 行任务）对相同的前沿模型在 23-59%。

**类型：** Learn（学习）
**语言：** Python（标准库，CodeAct vs JSON 工具调用对比）
**前置课程：** Phase 14 · 07（Tool use），Phase 15 · 01（Long-horizon agents）
**时间：** ~45 分钟

## 问题

"哪个编码智能体最好"是错误的问题。正确的问题是：在匹配我工作的任务分布上，用我将在生产中运行的脚手架，我能获得什么端到端可靠性？

2022 到 2026 年间，该领域学到了脚手架——检索层、规划器、沙箱、编辑-验证循环、反馈格式——是承重的。Claude Sonnet 4.5 在 SWE-agent v1 上在 SWE-bench Verified 上得分 43.2%；同一个模型在 Cline 的自主脚手架内得分 59.8%。16.6 个百分点的绝对差异，相同的权重。基础模型是一个组件；循环才是产品。

伴随的问题是基准饱和掩盖了退步。SWE-bench Verified 接近饱和，而简单任务尾巴（500 个任务中 161 个需要 ≤2 行）拉高了顶级分数。真实世界质量在 SWE-bench Pro（10+ 行更改）等分布上更好衡量，同样的领先者仍在 23-59%。

## 核心概念

### SWE-bench，一段话

SWE-bench（Jimenez 等）取真实的 GitHub issue 及其基准真相补丁，要求智能体产生使测试套件通过的补丁。SWE-bench Verified（OpenAI，2024）是一个人工策划的 500 任务子集，去除了模糊和损坏的任务。SWE-bench Pro 是更难的后继者——需要 10+ 行更改的任务，当前前沿智能体在 23-59%。

### 2022 → 2026 曲线实际展示了什么

- **2022**：研究模型在原始 SWE-bench 上约 4%。
- **2024**：GPT-4 + Devin 风格脚手架约 14%；SWE-agent 约 12%。
- **2025**：Claude 3.5/3.7 Sonnet 在 Aider 和 SWE-agent 内推进到 40-55% 范围。
- **2026**：Claude Sonnet 4.5 和前沿竞争者在 SWE-bench Verified 上 70-80%+。Epoch AI 的排行榜实时跟踪。

斜率来自三个复合来源：更好的基础模型、更好的脚手架（CodeAct、反思、验证器循环）、更好的基准（Verified 去除噪声）。

### CodeAct vs JSON 工具调用

OpenHands（All-Hands-AI，arXiv:2407.16741，前 OpenDevin）做了一个特定的架构赌注：模型不是发出由宿主解码和执行的 JSON 工具调用，而是发出 Python 代码，由 Jupyter 风格的内核在沙箱中运行。智能体可以在一个动作内循环文件、链接工具和捕获自己的异常。

权衡：

- **JSON 工具调用**：每个动作是一轮；易于审计；有限的组合性；默认安全，因为每次调用都通过显式验证器。
- **CodeAct**：一个动作可以是整个程序；组合性强；需要加固的沙箱（OpenHands 使用 Docker 隔离）；失败模式包括沙箱运行时允许的任何事情。

两种架构都在生产中。CodeAct 在开放平台（OpenHands、smolagents）中占主导。JSON 工具调用在托管服务（Anthropic Managed Agents、OpenAI Assistants）中仍占主导，提供商控制执行器。

### 2026 年版图中的脚手架

| 脚手架 | 许可证 | 执行模型 | 显著特性 |
|---|---|---|---|
| OpenHands (OpenDevin) | MIT | Docker 中的 CodeAct | 最活跃的开放平台；事件流可重放 |
| SWE-agent | MIT | 智能体-计算机接口（ACI） | 第一个端到端 SWE-bench 脚手架 |
| Aider | Apache-2 | 本地仓库中的 diff 编辑 | 最小脚手架，强回归稳定性 |
| Cline | Apache-2 | 带工具策略的 VS Code 智能体 | Sonnet 4.5 上得分最高的开放脚手架 |
| Devin (Cognition) | 专有 | 托管 VM + 规划器 | 第一个"AI 软件工程师"产品类别 |
| Claude Code | 专有 | 权限模式 + 例程 | Lesson 10 详细讲解智能体循环 |

### 为什么脚手架占主导

编码运行是一个长视界轨迹（Lesson 1）。可靠性跨步骤复合。脚手架得分的三个地方：

1. **检索**：找到正确的文件来读取是无声的瓶颈。SWE-agent 的 ACI、OpenHands 的文件索引和 Aider 的仓库图都针对此。
2. **验证器循环**：运行测试、读取堆栈跟踪和重试在 SWE-bench 上是 10+ 个百分点的差距。
3. **失败遏制**：在错误时回滚的沙箱防止复合损害。同一个模型有和没有验证器循环看起来像两个不同的产品。

### 基准饱和与真实分布

OpenHands 作者和 Epoch AI 都标记 SWE-bench Verified 有一个简单尾巴：500 个任务中 161 个只需要 1-2 行更改。高分部分由这个尾巴驱动。SWE-bench Pro 限制为 10+ 行更改，即使前沿系统也返回 23-59% 的分数。你的生产分布几乎肯定更接近 Pro 而非 Verified。

选择智能体的含义：在你自己的 bug 积压中运行一个类似 Pro 的子集。重要的分数是在代表你发布内容的任务上的分数。

## 动手实现

`code/main.py` 在固定的迷你任务分布上比较两个玩具智能体脚手架：

1. 一个 **JSON 工具调用**脚手架，每轮执行一个动作。
2. 一个 **CodeAct**脚手架，每个动作可以发出一个小的 Python 片段。

两者都使用存根"模型"（确定性规则），因此比较将脚手架与模型质量隔离。输出显示 CodeAct 脚手架以更大的每动作爆炸半径为代价在更少的轮次中解决更多任务。

## 实际应用

`outputs/skill-scaffold-audit.md` 帮助你在采用之前审计提议的编码智能体脚手架：检索质量、验证器存在、沙箱隔离和基准到分布的适配。

## 练习

1. 运行 `code/main.py`。每个脚手架在相同任务集上需要多少轮？每个的每动作爆炸半径是多少？

2. 阅读 OpenHands 论文（arXiv:2407.16741）。论文认为 CodeAct 在复杂任务上击败 JSON 工具调用。识别论文承认的一个失败模式，并写一句话说明该模式何时会在生产中占主导。

3. 从你的 bug 积压中选择一个需要跨两个文件 10+ 行更改的任务。估计前沿模型在（a）JSON 工具调用和（b）CodeAct 下的端到端成功概率。论证差距。

4. SWE-bench Verified 有 161 个单文件、1-2 行任务。构造一个排除它们的分数。排行榜如何重新排列？

5. 阅读"Introducing SWE-bench Verified"（OpenAI）。解释用于去除模糊任务的具体方法论，并命名策划会遗漏的一个类别。

## 关键术语

| 术语 | 常见说法 | 实际含义 |
|---|---|---|
| SWE-bench | "编码基准" | 带基准真相补丁和测试套件的真实 GitHub issue |
| SWE-bench Verified | "清理的子集" | 500 个人工策划的任务，存在简单尾巴 |
| SWE-bench Pro | "更难的子集" | 10+ 行更改；前沿在 23-59% |
| CodeAct | "代码即动作" | 智能体发出 Python；Jupyter 风格内核在沙箱中执行 |
| JSON tool call（JSON 工具调用） | "函数调用" | 每个动作是执行前验证的结构化 JSON 有效载荷 |
| Scaffold（脚手架） | "智能体框架" | 基础模型周围的检索 + 规划器 + 执行器 + 验证器循环 |
| ACI（智能体-计算机接口） | "SWE-agent 的格式" | 为 LLM 人体工学设计的命令集，而非人类 shell |
| Verifier loop（验证器循环） | "测试-重试" | 运行测试、读取输出、修订补丁；最大的非模型可靠性增益 |

## 延伸阅读

- [Jimenez 等 — SWE-bench](https://www.swebench.com/) — 原始基准和方法论
- [OpenAI — Introducing SWE-bench Verified](https://openai.com/index/introducing-swe-bench-verified/) — 策划子集如何构建
- [Wang 等 — OpenHands: An Open Platform for AI Software Developers](https://arxiv.org/abs/2407.16741) — CodeAct 架构和事件流设计
- [Epoch AI — SWE-bench leaderboard](https://epoch.ai/benchmarks) — 实时跟踪分数
- [Anthropic — Measuring agent autonomy](https://www.anthropic.com/research/measuring-agent-autonomy) — 长视界编码智能体可靠性框架