# Capstone 01 — 终端原生编码 Agent

> 到 2026 年，编码 Agent 的形态已经定型。一个 TUI 驱动框架、一份有状态计划、一个沙箱化的工具接口、一个"规划→行动→观察→恢复"的循环。Claude Code、Cursor 3 和 OpenCode 从 50 英尺外看几乎一模一样。本毕设要求你从零构建一个端到端的编码 Agent——CLI 输入，Pull Request 输出——并在 SWE-bench Pro 上与 mini-swe-agent 和 Live-SWE-agent 进行对比评测。你将体会到，难点不在于模型调用，而在于工具循环、沙箱隔离和 50 轮运行的成本天花板。

**类型：** 毕业设计
**语言：** TypeScript / Bun（框架）、Python（评测脚本）
**前置课程：** Phase 11（LLM 工程）、Phase 13（工具与协议）、Phase 14（Agent 工程）、Phase 15（自主系统）、Phase 17（基础设施与生产）
**涵盖阶段：** P0 · P5 · P7 · P10 · P11 · P13 · P14 · P15 · P17 · P18
**时间：** ~35 小时

## 问题描述

2026 年，编码 Agent 成为 AI 应用的主导品类。Claude Code（Anthropic）、Cursor 3 搭配 Composer 2 和 Agent Tabs（Cursor）、Amp（Sourcegraph）、OpenCode（11.2 万星）、Factory Droids 和 Google Jules 都在发布同一架构的变体：一个终端框架、一个受权限控制的工具接口、一个沙箱，以及围绕前沿模型构建的"规划-行动-观察"循环。前沿很窄——Live-SWE-agent 在 SWE-bench Verified 上用 Opus 4.5 达到了 79.2%——但工程宽度很大。大多数失败模式不是模型犯错，而是工具循环不稳定、上下文污染、Token 成本失控和破坏性文件系统操作。

你无法从外部理解这些 Agent。你必须亲手构建一个，亲眼看着循环在第 47 轮崩溃——因为 ripgrep 返回了 8MB 的匹配结果——然后重构截断层。这就是本毕设的意义。

## 核心概念

框架有四个接口面。**规划（Plan）** 维护一个 TodoWrite 风格的状态对象，模型每轮重写它。**行动（Act）** 分发工具调用（read、edit、run、search、git）。**观察（Observe）** 捕获 stdout / stderr / 退出码，截断后将摘要反馈给模型。**恢复（Recover）** 处理工具错误，避免上下文窗口溢出或无限循环。2026 年的形态增加了一个要素：**钩子（Hooks）**。`PreToolUse`、`PostToolUse`、`SessionStart`、`SessionEnd`、`UserPromptSubmit`、`Notification`、`Stop` 和 `PreCompact`——可配置的扩展点，运营者可在其中注入策略、遥测和防护栏。

沙箱使用 E2B 或 Daytona。每个任务在一个全新的 devcontainer 中运行，挂载一个可读写的 git worktree。框架从不接触宿主文件系统。worktree 在成功或失败后被销毁。成本控制在三层执行：每轮 Token 上限、每会话美元预算和硬性轮次上限（通常 50 轮）。可观测性层使用 OpenTelemetry span，遵循 GenAI 语义约定，发送到自托管的 Langfuse。

## 架构

```
  用户 CLI  →  框架（Bun + Ink TUI）
                   │
                   ↓
            规划 / 行动 / 观察循环  ←→  Claude Sonnet 4.7 / GPT-5.4-Codex / Gemini 3 Pro
                   │                          （通过 OpenRouter，模型无关）
                   ↓
            工具分发器（MCP StreamableHTTP 客户端）
                   │
      ┌────────────┼────────────┬──────────┐
      ↓            ↓            ↓          ↓
   read/edit    ripgrep     tree-sitter   git/run
      │            │            │          │
      └────────────┴────────────┴──────────┘
                   │
                   ↓
            E2B / Daytona 沙箱（worktree 隔离）
                   │
                   ↓
            钩子：Pre/Post、Session、Prompt、Compact
                   │
                   ↓
            OpenTelemetry → Langfuse（spans、tokens、$）
                   │
                   ↓
            通过 GitHub App 发起 PR
```

## 技术栈

- 框架运行时：Bun 1.2 + Ink 5（React-in-terminal）
- 模型访问：OpenRouter 统一 API，支持 Claude Sonnet 4.7、GPT-5.4-Codex、Gemini 3 Pro、Opus 4.5（用于最难的任务）
- 工具传输：Model Context Protocol StreamableHTTP（MCP 2026 修订版）
- 沙箱：E2B 沙箱（JS SDK）或 Daytona devcontainers
- 代码搜索：ripgrep 子进程、tree-sitter 解析器（17 种语言，预编译）
- 隔离：每个任务使用 `git worktree add` 创建独立分支，成功/失败后清理
- 评测框架：SWE-bench Pro（verified 子集）+ Terminal-Bench 2.0 + 你自己的 30 任务保留集
- 可观测性：OpenTelemetry SDK + `gen_ai.*` 语义约定 → 自托管 Langfuse
- PR 发布：GitHub App + 细粒度 Token，作用域限定在目标仓库

## 构建步骤

1. **TUI 和命令循环。** 用 Ink 搭建 Bun 项目。接受 `agent run <repo> "<task>"` 命令。打印分屏视图：计划面板（顶部）、工具调用流（中部）、Token 预算（底部）。添加 Ctrl-C 取消功能，退出前触发 `SessionEnd` 钩子。

2. **计划状态。** 定义类型化的 TodoWrite schema（pending / in_progress / done 条目及备注）。模型每轮以工具调用形式重写完整状态——不允许增量修改。将计划持久化到 `.agent/state.json`，以便崩溃后恢复。

3. **工具接口。** 定义六个工具：`read_file`、`edit_file`（带 diff 预览）、`ripgrep`、`tree_sitter_symbols`、`run_shell`（带超时）、`git`（status / diff / commit / push）。通过 MCP StreamableHTTP 暴露，使框架与传输层无关。每个工具返回截断的输出（每次调用上限 4k token）。

4. **沙箱封装。** 每个任务启动一个 E2B 沙箱。使用 `git worktree add -b agent/$TASK_ID` 创建新分支。所有工具调用在沙箱内执行。宿主文件系统不可达。

5. **钩子。** 实现所有 8 种 2026 年钩子类型。至少接入 4 个用户自定义钩子：(a) `PreToolUse` 破坏性命令守卫，阻止在 worktree 外执行 `rm -rf`；(b) `PostToolUse` Token 记账；(c) `SessionStart` 预算初始化；(d) `Stop` 写入最终 trace 包。

6. **评测循环。** 克隆 SWE-bench Pro Python 的 30 个 issue 子集。对每个任务运行你的框架。与 mini-swe-agent（最小基线）在 pass@1、每任务轮次和每任务成本上进行对比。将结果写入 `eval/results.jsonl`。

7. **成本控制。** 硬性截断：50 轮、200k 上下文、每任务 $5。`PreCompact` 钩子在 150k 标记处将早期轮次摘要为先前状态块，释放空间容纳新观察而不丢失计划。

8. **PR 发布。** 成功后，最后一步是 `git push` + GitHub API 调用，打开一个 PR，正文包含计划和 diff 摘要。

## 使用示例

```
$ agent run ./my-repo "修复 worker.rs 中的竞态条件"
[plan]  1 定位 worker.rs 并枚举 mutex 使用
        2 识别竞争下的共享状态
        3 提出修复方案，验证测试
[tool]  ripgrep mutex.*lock -t rust           （44 处匹配，已截断）
[tool]  read_file src/worker.rs 120..180
[tool]  edit_file src/worker.rs (+8 -3)
[tool]  run_shell cargo test worker::          （通过）
[plan]  1 完成 · 2 完成 · 3 完成
[done]  PR 已创建：#482   轮次=9   tokens=38k   成本=$0.41
```

## 交付物

可复用技能文件位于 `outputs/skill-terminal-coding-agent.md`。给定仓库路径和任务描述，在沙箱中运行完整的规划-行动-观察循环，返回 PR URL 和 trace 包。评分标准：

| 权重 | 评分项 | 衡量方式 |
|:-:|---|---|
| 25 | SWE-bench Pro pass@1 vs 基线 | 你的框架 vs mini-swe-agent，在 30 个匹配的 Python 任务上对比 |
| 20 | 架构清晰度 | Plan/Act/Observe 分离、钩子接口、工具 schema——参照 Live-SWE-agent 布局评审 |
| 20 | 安全性 | 沙箱逃逸测试、权限提示、破坏性命令守卫通过红队测试 |
| 20 | 可观测性 | Trace 完整性（100% 工具调用被 span 覆盖）、每轮 Token 记账 |
| 15 | 开发者体验 | 冷启动 < 2s、崩溃恢复可续接计划、Ctrl-C 可干净取消进行中的工具调用 |
| **100** | | |

## 练习

1. 将底层模型从 Claude Sonnet 4.7 切换为在 vLLM 上运行的 Qwen3-Coder-30B。对比 pass@1 和每任务成本。报告开源模型在哪些方面表现不足。

2. 添加一个 `reviewer` 子 Agent，在 PR 发布前阅读 diff，可请求修订循环。衡量误报的审查是否会将 SWE-bench 通过率降低到单 Agent 基线以下（提示：通常会）。

3. 压力测试沙箱：编写一个尝试 `curl` 外部 URL 的任务和一个尝试写入 worktree 外目录的任务。确认两者都被 PreToolUse 钩子阻止。记录尝试日志。

4. 使用较小的模型（Haiku 4.5）实现 `PreCompact` 摘要。衡量 3 倍压缩时计划保真度损失了多少。

5. 将 MCP StreamableHTTP 传输切换为 stdio。基准测试冷启动和单次调用延迟。为本地使用场景选择最优方案。

## 关键术语

| 术语 | 业界说法 | 实际含义 |
|------|---------|---------|
| 框架（Harness） | "Agent 循环" | 包围模型的代码层，负责分发工具、维护计划状态和执行预算约束 |
| 钩子（Hook） | "Agent 事件监听器" | 用户编写的脚本，在框架的 8 个生命周期事件之一触发时运行 |
| Worktree | "Git 沙箱" | 链接到单独路径的 git 检出副本；可丢弃而不影响主克隆 |
| TodoWrite | "计划状态" | 模型每轮重写的类型化列表，包含 pending/in_progress/done 条目 |
| StreamableHTTP | "MCP 传输" | 2026 年 MCP 修订版：长连接 + 双向流；替代 SSE |
| Token 上限 | "上下文预算" | 每轮或每会话的输入+输出 token 上限；触发压缩或终止 |
| pass@1 | "单次尝试通过率" | SWE-bench 任务中首次运行即通过、无重试或测试集窥探的比例 |

## 延伸阅读

- [Claude Code 文档](https://docs.anthropic.com/en/docs/claude-code) — Anthropic 的参考框架
- [Cursor 3 更新日志](https://cursor.com/changelog) — Agent Tabs 和 Composer 2 产品说明
- [mini-swe-agent](https://github.com/SWE-agent/mini-swe-agent) — SWE-bench 框架对比的最小基线
- [Live-SWE-agent](https://github.com/OpenAutoCoder/live-swe-agent) — 用 Opus 4.5 在 SWE-bench Verified 上达到 79.2%
- [OpenCode](https://opencode.ai) — 开源框架，11.2 万星
- [SWE-bench Pro 排行榜](https://www.swebench.com) — 本毕设的评测目标
- [Model Context Protocol 2026 路线图](https://blog.modelcontextprotocol.io/posts/2026-mcp-roadmap/) — StreamableHTTP、能力元数据
- [OpenTelemetry GenAI 语义约定](https://opentelemetry.io/docs/specs/semconv/gen-ai/) — 工具调用和 Token 用量的 span schema