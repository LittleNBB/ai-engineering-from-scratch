# Capstone 09 — 代码迁移 Agent（仓库级语言/运行时升级）

> Amazon 的 MigrationBench（Java 8 到 17）和 Google 的 App Engine Py2-to-Py3 迁移器设定了 2026 年的标杆。Moderne 的 OpenRewrite 能大规模进行确定性 AST 重写。Grit 用 codemod 风格的 DSL 解决同一问题。生产模式将两者结合：确定性底层处理安全重写，Agent 层处理模糊情况，沙箱用于逐分支构建，测试框架在 PR 发布前确保全绿。本毕设要求你迁移 50 个真实仓库，发布通过率和失败分类法。

**类型：** 毕业设计
**语言：** Python（Agent）、Java / Python（迁移目标）、TypeScript（仪表板）
**前置课程：** Phase 5（NLP）、Phase 7（Transformer）、Phase 11（LLM 工程）、Phase 13（工具）、Phase 14（Agent 工程）、Phase 15（自主系统）、Phase 17（基础设施与生产）
**涵盖阶段：** P5 · P7 · P11 · P13 · P14 · P15 · P17
**时间：** ~30 小时

## 问题描述

大规模代码迁移是 2026 年编码 Agent 最清晰的生产应用场景之一。真值显而易见（迁移后测试套件是否通过？），回报真实存在（Java 8 集群迁移是人力规模的项目），基准公开可用（MigrationBench 50 仓库子集）。Moderne 的 OpenRewrite 处理确定性侧。Agent 层处理 OpenRewrite 配方无法覆盖的一切：模糊重写、构建系统漂移、长尾语法、传递性依赖破坏。

你将构建一个 Agent，接收一个 Java 8 仓库（或 Python 2 仓库），产出一个 CI 全绿的迁移分支。你将衡量通过率、测试覆盖率保持度、每仓库成本，并构建一个失败分类法。与纯确定性基线的对比将告诉你 Agent 的价值究竟在哪里。

## 核心概念

管线有两层。**确定性底层**（Java 用 OpenRewrite，Python 用 libcst）安全地执行大部分机械化重写：导入、方法签名、空安全编辑、try-with-resources、废弃 API 替换。它速度快且产出可审计的 diff。**Agent 层**（OpenAI Agents SDK 或 LangGraph，搭配 Claude Opus 4.7 和 GPT-5.4-Codex）处理配方无法覆盖的情况：构建文件升级（Maven/Gradle/pyproject）、传递性依赖冲突、测试不稳定、自定义注解。

每个仓库获得一个预装目标运行时的 Daytona 沙箱。Agent 迭代：运行构建、分类失败、应用修复、重跑。硬性限制：每仓库 30 分钟、$8、20 轮 Agent 调用。如果所有测试通过且覆盖率增量非负，分支自动开 PR。否则，仓库被归入某个失败类别并附带证据。

失败分类法是交付物。50 个仓库中，什么坏了？传递性依赖？自定义注解？构建工具版本？与迁移无关的测试不稳定？每个类别有计数和示例 diff。未来的配方作者可以针对前三名发力。

## 架构

```
目标仓库
      │
      ↓
OpenRewrite / libcst 确定性配方
   （安全、快速、可审计，覆盖 ~70-80% 修复）
      │
      ↓
Daytona 沙箱（每分支一个）
      │
      ↓
Agent 循环（Claude Opus 4.7 / GPT-5.4-Codex）：
   - 运行构建 -> 捕获失败
   - 分类失败（构建、测试、lint）
   - 应用修复（打补丁或重跑配方）
   - 重跑
   - 预算：30 分钟、$8、20 轮
      │
      ↓
测试 + 覆盖率增量门
      │
      ↓ （通过）
开 PR
      │
      ↓ （失败）
归入失败类别 + 附带复现材料
```

## 技术栈

- 确定性底层：OpenRewrite（Java）或 libcst（Python）
- Agent：OpenAI Agents SDK 或 LangGraph，搭配 Claude Opus 4.7 + GPT-5.4-Codex
- 沙箱：Daytona devcontainer（每分支一个），预装目标运行时（Java 17 / Python 3.12）
- 构建系统：Maven、Gradle、uv（Python）
- 基准：Amazon MigrationBench 50 仓库子集（Java 8 到 17）、Google App Engine Py2-to-Py3 仓库
- 测试框架：并行运行器，Jacoco（Java）或 coverage.py（Python）覆盖率
- 可观测性：Langfuse + 每仓库 trace 包（含每个 diff 块）
- 仪表板：失败分类法仪表板，含每类计数和示例 diff

## 构建步骤

1. **配方遍历。** 先运行 OpenRewrite（Java）或 libcst（Python）配方。捕获 70-80% 的机械化迁移。提交为 "recipe" commit。

2. **构建试运行。** Daytona 沙箱：安装目标运行时，运行构建。如果全绿，跳到测试。如果红灯，交给 Agent。

3. **Agent 循环。** LangGraph 配合工具：`run_build`、`read_file`、`edit_file`、`run_test`、`git_diff`。Agent 分类失败（依赖、语法、测试、构建工具）并应用针对性修复。重跑。

4. **预算上限。** 每仓库 30 分钟墙上时间、$8 成本、20 轮 Agent 调用。任何超限都停止并归入 "budget_exhausted"，附带当前 diff。

5. **测试 + 覆盖率门。** 构建全绿后，运行测试套件。与基础仓库对比覆盖率。如果覆盖率下降超过 2%，归入 "coverage_regression"。

6. **开 PR。** 成功后，推送分支，打开 PR，正文包含 diff 和摘要（哪些配方生效、哪些 commit 由 Agent 编写）。

7. **失败分类法。** 每个失败仓库打标签：`dep_upgrade_required`、`build_tool_drift`、`custom_annotation`、`test_flake`、`syntax_edge_case`、`budget_exhausted`。构建仪表板。

8. **50 仓库运行。** 在 MigrationBench 子集上执行。报告每类通过率、每仓库成本、覆盖率保持度，以及与纯确定性基线的对比。

## 使用示例

```
$ migrate legacy-java-service --target java17
[recipe]   应用了 27 个重写（JUnit 4->5、HashMap 初始化、try-with-resources）
[build]    失败：找不到符号 sun.misc.BASE64Encoder
[agent]    第 1 轮 分类：removed_jdk_api
[agent]    第 2 轮 应用：sun.misc.BASE64Encoder -> java.util.Base64
[build]    通过
[tests]    412/412 通过；覆盖率 84.1% -> 84.3%
[pr]       已开 #1841  成本=$3.20  轮次=4
```

## 交付物

`outputs/skill-migration-agent.md` 是交付物。给定一个仓库，执行确定性配方后接 Agent 循环，产出 CI 全绿的迁移分支，或将仓库归入分类法类别。

| 权重 | 评分项 | 衡量方式 |
|:-:|---|---|
| 25 | MigrationBench 通过率 | 50 仓库子集 pass@1 |
| 20 | 测试覆盖率保持度 | 与基础仓库的平均覆盖率增量 |
| 20 | 每迁移仓库成本 | 通过运行的 $/仓库 |
| 20 | Agent/确定性工具集成 | OpenRewrite 处理 vs Agent 编写的修复比例 |
| 15 | 失败分析报告 | 分类法完整性及示例 |
| **100** | | |

## 练习

1. 仅用 OpenRewrite（无 Agent）运行迁移管线。与完整管线对比通过率。识别 Agent 单独做出贡献的场景。

2. 实现 "lint-clean" 检查：迁移后运行代码风格 linter（Java 用 spotless，Python 用 ruff）。如果出现新 lint 错误则 PR 失败。衡量覆盖率保持但风格退化的比例。

3. 添加 "minimal-diff" 优化器：Agent 分支通过测试后，用第二轮修剪不必要的变更。报告 diff 大小缩减。

4. 扩展到第三种迁移：Node 18 到 Node 22。复用沙箱封装；替换配方层为自定义 codemod。

5. 衡量首次全绿构建时间（TTFGB）作为 UX 指标。目标：p50 低于 10 分钟。

## 关键术语

| 术语 | 业界说法 | 实际含义 |
|------|---------|---------|
| 确定性底层 | "配方引擎" | OpenRewrite / libcst：带安全保证的声明式 AST 重写 |
| Codemod | "代码修改程序" | 机械化变更源代码的重写规则 |
| 构建漂移 | "工具版本偏差" | Maven / Gradle / uv 主版本间的细微行为变化 |
| 失败类别 | "分类桶" | 仓库未迁移的标记原因：依赖、语法、测试、构建工具、预算 |
| 覆盖率增量 | "覆盖率保持度" | 从基础到迁移分支的测试覆盖率 % 变化 |
| Agent 轮次 | "工具调用轮" | Agent 循环中的一次 规划 -> 行动 -> 观察 |
| 预算耗尽 | "触顶" | 仓库在 30 分钟 / $8 / 20 轮限制内未通过 |

## 延伸阅读

- [Amazon MigrationBench](https://aws.amazon.com/blogs/devops/amazon-introduces-two-benchmark-datasets-for-evaluating-ai-agents-ability-on-code-migration/) — 2026 年标准基准
- [Moderne.io OpenRewrite 平台](https://www.moderne.io) — 确定性底层参考
- [OpenRewrite 文档](https://docs.openrewrite.org) — 配方编写
- [Grit.io](https://www.grit.io) — 备选 codemod DSL
- [OpenAI 沙箱化迁移 cookbook](https://developers.openai.com/cookbook/examples/agents_sdk/sandboxed-code-migration/sandboxed_code_migration_agent) — Agents SDK 参考
- [Google App Engine Py2 到 Py3 迁移器](https://cloud.google.com/appengine) — 备选迁移基准
- [libcst](https://github.com/Instagram/LibCST) — Python 确定性底层
- [Daytona 沙箱](https://daytona.io) — 参考级逐分支沙箱