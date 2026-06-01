# Capstone 16 — GitHub Issue-to-PR 自主 Agent

> AWS Remote SWE Agents、Cursor Background Agents、OpenAI Codex cloud 和 Google Jules 在 2026 年发布了同一产品形态：给 issue 打标签，得到一个 PR。在云沙箱中运行 Agent，验证测试通过，发布带理由的可评审 PR。难点在于自动复现仓库的构建环境、防止凭据泄漏、执行每仓库预算、确保 Agent 无法强制推送。本毕设构建自托管版本，在成本和通过率上与托管方案进行对比。

**类型：** 毕业设计
**语言：** Python（Agent）、TypeScript（GitHub App）、YAML（Actions）
**前置课程：** Phase 11（LLM 工程）、Phase 13（工具）、Phase 14（Agent 工程）、Phase 15（自主系统）、Phase 17（基础设施与生产）
**涵盖阶段：** P11 · P13 · P14 · P15 · P17
**时间：** ~30 小时

## 问题描述

异步云编码 Agent 是独立于交互式编码 Agent（Capstone 01）的产品品类。UX 是一个 GitHub 标签。你给 issue 打上 `@agent fix this`，一个 worker 在云沙箱中启动，克隆仓库，运行测试，编辑文件，验证，然后打开一个 PR，正文包含 Agent 的理由。没有交互循环，没有终端。AWS Remote SWE Agents、Cursor Background Agents、OpenAI Codex cloud、Google Jules 和 Factory Droids 都收敛于此。

工程挑战是具体的：环境复现（Agent 必须从零构建仓库，无缓存开发镜像）、不稳定测试（必须重跑或隔离）、凭据作用域（最小细粒度权限的 GitHub App）、每仓库每日预算执行、以及禁止强制推送策略。本毕设衡量通过率、成本和安全性，与托管方案对比。

## 核心概念

触发器是 GitHub webhook（issue 标签或 PR 评论）。分发器将任务入队到 ECS Fargate 或 Lambda。Worker 将仓库拉入 Daytona 或 E2B 沙箱，配有一个从仓库推断的通用 Dockerfile（语言、框架）。Agent 对 Claude Opus 4.7 或 GPT-5.4-Codex 运行 mini-swe-agent 或 SWE-agent v2 循环。迭代：读代码、提出修复、应用补丁、运行测试。

验证是门控步骤。PR 发布前，完整 CI 必须在沙箱中通过。计算覆盖率增量；如果超出阈值为负，PR 发布但标记 `needs-review`。Agent 将理由作为 PR 描述发布，加上 `@agent` 线程供评审者后续跟进。

安全通过两个不同的 GitHub 面作用域化：App 提供短生命周期的 installation token，具有 `workflows: read` 和窄范围的 repo contents/PR 权限；分支保护（非 App 权限）强制"不直接写入 `main`"和"禁止强制推送"——App 从不被添加到绕过列表中。对 `.github/workflows` 的路径作用域只读访问不是真正的 GitHub App 原语，因此 Agent 的文件编辑允许列表必须在 worker 端强制执行。每仓库每日预算上限在分发器执行（例如，每仓库每日最多 5 个 PR，每个 PR $20）。

## 架构

```
GitHub issue 标记 `@agent fix` 或 PR 评论
            │
            ↓
    GitHub App webhook -> AWS Lambda 分发器
            │
            ↓
    ECS Fargate 任务（或 GitHub Actions 自托管 runner）
       - 拉取仓库
       - 推断 Dockerfile（语言、包管理器）
       - Daytona / E2B 沙箱 + 目标运行时
       - clone -> git worktree -> Agent 分支
            │
            ↓
    mini-swe-agent / SWE-agent v2 循环
       Claude Opus 4.7 或 GPT-5.4-Codex
       工具：ripgrep、tree-sitter、read/edit、run_tests、git
            │
            ↓
    验证 CI 在沙箱中通过 + 覆盖率增量检查
            │
            ↓ （已验证）
    git push + 通过 GitHub App 开 PR
       PR 正文 = 理由 + diff 摘要 + trace URL
       标签：needs-review
            │
            ↓
    运营者评审；可 @提及 Agent 进行后续跟进
```

## 技术栈

- 触发：GitHub App + 细粒度 token；webhook 接收器通过 Lambda 或 Fly.io
- Worker：ECS Fargate 任务（或 GitHub Actions 自托管 runner）
- 沙箱：每任务一个 Daytona devcontainer 或 E2B 沙箱
- Agent 循环：mini-swe-agent 基线或 SWE-agent v2，搭配 Claude Opus 4.7 / GPT-5.4-Codex
- 检索：tree-sitter 仓库地图 + ripgrep
- 验证：沙箱内完整 CI + 覆盖率增量门
- 可观测性：Langfuse，每 PR trace 归档，从 PR 正文链接
- 预算：每仓库每日美元上限；每仓库每日最大 PR 数

## 构建步骤

1. **GitHub App。** 细粒度 installation token：issues 读+写、pull_requests 写、contents 读+写、workflows 读。分支保护（唯一能执行此功能的面）强制"不直接推送到 `main`"和"禁止强制推送"；App 不在绕过列表中。Worker 以允许列表检查方式强制"不写入 `.github/workflows` 下的文件"，因为 GitHub App 权限不是路径作用域的。

2. **Webhook 接收器。** Lambda 函数接受 issue 标签 / PR 评论 webhook。按标签 `@agent fix this` 过滤。入队到 SQS。

3. **分发器。** 从 SQS 弹出任务。执行每仓库每日预算。启动 ECS Fargate 任务，含仓库 URL、issue 正文和新鲜 Daytona 沙箱。

4. **环境推断。** 检测语言（Python、Node、Go、Rust）和包管理器（uv、pnpm、go mod、cargo）。如果不存在则动态生成 Dockerfile。

5. **Agent 循环。** mini-swe-agent 或 SWE-agent v2 搭配 Claude Opus 4.7。工具：ripgrep、tree-sitter repo-map、read_file、edit_file、run_tests、git。硬性限制：$20 成本、30 分钟墙上时间、30 轮 Agent 调用。

6. **验证。** 循环结束后，在沙箱中运行完整测试套件。通过 jacoco / coverage.py 计算覆盖率增量。如果 CI 红灯：停止，不开 PR。如果覆盖率下降超过 2%：开 PR 并标记 `needs-review`。

7. **PR 发布。** 推送 Agent 分支。通过 GitHub API 开 PR，含：标题、理由、diff 摘要、trace URL、成本、轮次。

8. **凭据卫生。** Worker 使用短生命周期的 GitHub App installation token 运行。归档前清洗日志中的密钥。

9. **评测。** 30 个不同难度的种子内部 issue。衡量通过率、PR 质量（diff 大小、风格、覆盖率）、成本、延迟。在相同 issue 上与 Cursor Background Agents 和 AWS Remote SWE Agents 对比。

## 使用示例

```
# 在 github.com 上
  - 用户给 issue #842 打上 `@agent fix this` 标签
  - 14 分钟后 PR #1903 出现
  - 正文：
    > 修复了 widget.dedupe() 中由空比较器条目引起的 NPE。
    > 添加了回归测试 widget_test.go::TestDedupeNullComparator。
    > 覆盖率增量：+0.12%
    > 轮次：7  成本：$1.80  Trace：langfuse:...
    > 标签：needs-review
```

## 交付物

`outputs/skill-issue-to-pr.md` 是交付物。GitHub App + 异步云 worker，将标记的 issue 转化为可评审的 PR，具有有界成本和作用域化凭据。

| 权重 | 评分项 | 衡量方式 |
|:-:|---|---|
| 25 | 30 个 issue 的通过率 | 端到端成功（CI 全绿 + 覆盖率 OK） |
| 20 | PR 质量 | Diff 大小、覆盖率增量、风格一致性 |
| 20 | 每解决 issue 的成本和延迟 | 每 PR 的 $ 和墙上时间 |
| 20 | 安全性 | 作用域化 token、每仓库预算、禁止强制推送、凭据卫生 |
| 15 | 运营者体验 | 理由评论、重试能力、@提及后续跟进 |
| **100** | | |

## 练习

1. 添加"修复不稳定测试"模式：标签 `@agent stabilize-flake TestX` 在沙箱中运行测试 50 次，提出稳定它的最小变更。

2. 在三个共享 issue 上与 Cursor Background Agents 对比成本。报告哪些工具在哪里胜出。

3. 实现预算仪表板：每仓库每日成本、每用户成本。异常时告警。

4. 构建"dry-run"模式：不开 CI 而是打开草稿 PR，让评审者低成本审查计划。

5. 添加保留策略：7 天未合并的 PR 分支自动删除。

## 关键术语

| 术语 | 业界说法 | 实际含义 |
|------|---------|---------|
| GitHub App | "作用域化机器人身份" | 具有细粒度权限 + 短生命周期 installation token 的 App |
| 异步云 Agent | "后台 Agent" | 在云沙箱而非终端中运行的非交互式 worker |
| 环境推断 | "Dockerfile 合成" | 检测语言 + 包管理器，不存在时生成 Dockerfile |
| 验证 | "沙箱内 CI" | 开 PR 前在 worker 内运行完整测试套件 |
| 覆盖率增量 | "覆盖率保持度" | 从基础到 Agent 分支的测试覆盖率 % 变化 |
| 每仓库预算 | "每日上限" | 在分发器执行的美元和 PR 数上限 |
| 理由 | "PR 正文说明" | Agent 对变更内容和原因的摘要；PR 正文中必须包含 |

## 延伸阅读

- [AWS Remote SWE Agents](https://github.com/aws-samples/remote-swe-agents) — 标准异步云 Agent 参考
- [SWE-agent](https://github.com/SWE-agent/SWE-agent) — CLI 参考
- [Cursor Background Agents](https://docs.cursor.com/background-agent) — 商业备选方案
- [OpenAI Codex（cloud）](https://openai.com/codex) — 托管竞争方案
- [Google Jules](https://jules.google) — Google 的托管版本
- [Factory Droids](https://www.factory.ai) — 备选商业参考
- [GitHub App 文档](https://docs.github.com/en/apps) — 作用域化机器人身份
- [Daytona 云沙箱](https://daytona.io) — 参考级沙箱