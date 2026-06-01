# Capstone 10 — 多智能体软件工程团队

> SWE-AF 的工厂架构、MetaGPT 的角色化提示、AutoGen 0.4 的类型化 Actor 图、Cognition 的 Devin 和 Factory 的 Droids 在 2026 年收敛到同一形态：一个架构师规划、N 个编码者在并行 worktree 中工作、一个评审者把关、一个测试者验证。并行 worktree 将挂钟时间转化为吞吐量。共享状态和交接协议成为故障面。本毕设要求你构建这支团队，在 SWE-bench Pro 上评测，报告哪些交接会断裂以及频率。

**类型：** 毕业设计
**语言：** Python / TypeScript（Agent）、Shell（worktree 脚本）
**前置课程：** Phase 11（LLM 工程）、Phase 13（工具）、Phase 14（Agent 工程）、Phase 15（自主系统）、Phase 16（多智能体）、Phase 17（基础设施与生产）
**涵盖阶段：** P11 · P13 · P14 · P15 · P16 · P17
**时间：** ~40 小时

## 问题描述

单 Agent 编码框架在大型任务上触顶。不是因为单个 Agent 薄弱，而是因为 200k token 的上下文无法同时容纳架构计划、四份并行代码切片、评审意见和测试输出。多智能体工厂拆分问题：架构师拥有计划，编码者在并行 worktree 中独立实现子任务，评审者把关，测试者验证。SWE-AF 的"工厂"架构、MetaGPT 的角色、AutoGen 的类型化 Actor 图——三种表述描述同一形态。

故障面是交接。架构师规划了编码者无法实现的方案。编码者产出冲突的 diff。评审者批准了幻觉修复。测试者与仍在编码的编码者竞争。你将构建其中一支团队，在 50 个 SWE-bench Pro issue 上运行，追踪每次交接，发布事后分析报告。

## 核心概念

角色是类型化的 Agent。**架构师**（Claude Opus 4.7）读取 issue，撰写计划，拆分为带显式接口的子任务。**编码者**（Claude Sonnet 4.7，N 个并行实例，每个在 `git worktree` + Daytona 沙箱中）独立实现子任务。**评审者**（GPT-5.4）阅读合并后的 diff，要么批准要么请求具体修改。**测试者**（Gemini 2.5 Pro）在隔离环境中运行测试套件，报告通过/失败及制品。

通信通过共享任务板（文件或 Redis 支持）。每个角色订阅它被允许处理的消息。交接是 A2A 协议类型化消息。协调关注点：合并冲突解决（协调者角色或自动三方合并）、共享状态同步（计划一旦编码者启动就冻结；重新规划是独立事件）、评审者把关（评审者不能批准自己编写或提议的变更）。

Token 放大是隐性成本。每个角色边界都增加摘要提示和交接上下文。40 轮单 Agent 运行变成 4 个角色共 160 轮。评分标准专门衡量 Token 效率与单 Agent 基线的对比，因为问题不是"多 Agent 是否有效"而是"每美元是否赢了"。

## 架构

```
GitHub issue URL
      │
      ↓
架构师（Opus 4.7）
   读取 issue，产出带子任务 + 接口的计划
      │
      ↓
任务板（文件 / Redis）
      │
   ┌── 子任务 1 ──┬── 子任务 2 ──┬── 子任务 3 ──┬── 子任务 4 ──┐
   ↓              ↓              ↓              ↓              ↓
编码者 A       编码者 B       编码者 C       编码者 D       （4 个并行）
 (Sonnet)       (Sonnet)       (Sonnet)       (Sonnet)
 worktree A     worktree B     worktree C     worktree D
 Daytona        Daytona        Daytona        Daytona
      │              │              │              │
      └──────┬───────┴──────┬───────┴──────┘
             ↓
         合并协调者（三方合并 + 冲突解决）
             │
             ↓
         评审者（GPT-5.4）
             │
             ↓
         测试者（Gemini 2.5 Pro）-> 通过？-> 开 PR
                                    -> 失败？-> 路由回编码者
```

## 技术栈

- 编排：LangGraph，共享状态 + 每角色子图
- 消息：A2A 协议（Google 2025），类型化 Agent 间消息
- 模型：Opus 4.7（架构师）、Sonnet 4.7（编码者）、GPT-5.4（评审者）、Gemini 2.5 Pro（测试者）
- Worktree 隔离：每编码者 `git worktree add` + Daytona 沙箱
- 合并协调者：自定义三方合并 + LLM 介导的冲突解决
- 评测：SWE-bench Pro（50 个 issue）、SWE-AF 场景、HumanEval++ 单元测试
- 可观测性：Langfuse，角色标记的 span，每 Agent Token 记账
- 部署：K8s，每个角色独立 Deployment + 基于积压的 HPA

## 构建步骤

1. **任务板。** 文件支持的 JSONL，类型化消息：`plan_request`、`subtask`、`diff_ready`、`review_needed`、`test_needed`、`approved`、`rejected`、`replan_needed`。Agent 订阅标签。

2. **架构师。** 读取 GitHub issue，用 Opus 4.7 运行计划模板，要求显式子任务接口（涉及的文件、公共函数、测试影响）。发出一个 `plan_request`，包含子任务 DAG。

3. **编码者。** N 个并行工人，每个从任务板认领一个子任务。每个启动一个新的 `git worktree add` 分支加 Daytona 沙箱。实现子任务。发出 `diff_ready`，包含补丁 + 测试增量。

4. **合并协调者。** 所有编码者完成后，将 N 个分支三方合并到暂存分支。仅在文件级重叠时使用 LLM 介导的冲突解决。

5. **评审者。** GPT-5.4 阅读合并后的 diff。不能批准自己编写的 diff。发出 `approved`（无操作）或 `review_feedback`（带具体修改请求，路由回相关编码者）。

6. **测试者。** Gemini 2.5 Pro 在干净沙箱中运行测试套件。捕获制品。发出 `test_passed` 或 `test_failed`（含堆栈跟踪）。失败测试循环回拥有该子任务的编码者。

7. **交接记账。** 每条跨越角色边界的消息在 Langfuse 中获得一个 span，包含负载大小和使用的模型。计算每子任务 Token 放大（编码者_tokens + 评审者_tokens + 测试者_tokens + 架构师_share / 编码者_tokens）。

8. **评测。** 在 50 个 SWE-bench Pro issue 上运行。与单 Agent 基线（单个 Sonnet 4.7 在单 worktree 中）对比 pass@1 和每解决 issue 成本。

9. **事后分析。** 对每个失败 issue，识别断裂的交接（计划太模糊、合并冲突、评审误批、测试不稳定）。产出交接失败直方图。

## 使用示例

```
$ team run --issue https://github.com/acme/widget/issues/842
[architect] 计划：4 个子任务（parser、cache、api、migration）
[board]     分发给 4 个编码者，在并行 worktree 中
[coder-A]   子任务 parser  -> 42 行，本地测试通过
[coder-B]   子任务 cache   -> 88 行，本地测试通过
[coder-C]   子任务 api     -> 31 行，本地测试通过
[coder-D]   子任务 migration -> 19 行，本地测试通过
[merge]     三方合并：0 个冲突
[reviewer]  对 cache 评论（线程池大小）；路由到 coder-B
[coder-B]   修订：92 行；提交
[reviewer]  批准
[tester]    全部 412 个测试通过
[pr]        已开 #3382   4 个编码者，1 次修订，$4.90，18 分钟
```

## 交付物

`outputs/skill-multi-agent-team.md` 是交付物。给定一个 issue URL 和并行度，团队产出一个合并就绪的 PR，含每角色 Token 记账。

| 权重 | 评分项 | 衡量方式 |
|:-:|---|---|
| 25 | SWE-bench Pro pass@1 | 匹配的 50 issue 子集，pass@1 |
| 20 | 并行加速比 | 挂钟时间 vs 单 Agent 基线 |
| 20 | 评审质量 | 注入 bug 探针的误批准率 |
| 20 | Token 效率 | 每解决 issue 的总 Token vs 单 Agent |
| 15 | 协调工程 | 合并冲突解决、交接失败直方图 |
| **100** | | |

## 练习

1. 在运行中向某个 diff 注入一个明显 bug（在主体前加 `return None`）。衡量评审者的误批准率。调整评审者提示直到误批准率低于 5%。

2. 减少到两个编码者（架构师 + 编码者 + 评审者 + 测试者，编码者顺序运行两个子任务）。对比挂钟时间和通过率。

3. 用单写者约束替换合并协调者（子任务触及不相交的文件集）。衡量架构师的规划负担。

4. 将评审者从 GPT-5.4 换为 Claude Opus 4.7。衡量误批准率和 Token 成本差异。

5. 添加第五个角色：文档编写者（Haiku 4.5）。评审后生成 changelog 条目。衡量文档质量是否值得额外 Token 花费。

## 关键术语

| 术语 | 业界说法 | 实际含义 |
|------|---------|---------|
| 并行 worktree | "隔离分支" | `git worktree add` 为每个编码者生成独立工作树 |
| 任务板 | "共享消息总线" | 文件或 Redis 存储的类型化消息，Agent 订阅 |
| 交接 | "角色边界" | 从一个角色上下文跨越到另一个的任何消息 |
| Token 放大 | "多智能体开销" | 跨角色总 Token / 同任务的单 Agent Token |
| A2A 协议 | "Agent 间协议" | Google 2025 年的类型化 Agent 间消息规范 |
| 合并协调者 | "集成者" | 运行三方合并并调解冲突的组件 |
| 误批准 | "评审幻觉" | 评审者批准了含已知 bug 的 diff |

## 延伸阅读

- [SWE-AF 工厂架构](https://github.com/Agent-Field/SWE-AF) — 2026 年参考级多智能体工厂
- [MetaGPT](https://github.com/FoundationAgents/MetaGPT) — 角色化多智能体框架
- [AutoGen v0.4](https://github.com/microsoft/autogen) — 微软的类型化 Actor 框架
- [Cognition AI（Devin）](https://cognition.ai) — 参考级产品
- [Factory Droids](https://www.factory.ai) — 备选参考产品
- [Google A2A 协议](https://developers.google.com/agent-to-agent) — Agent 间消息规范
- [git worktree 文档](https://git-scm.com/docs/git-worktree) — 隔离底层
- [SWE-bench Pro](https://www.swebench.com) — 评测目标