# Capstone 15 — 宪法安全框架 + 红队靶场

> Anthropic 的 Constitutional Classifiers、Meta 的 Llama Guard 4、Google 的 ShieldGemma-2、NVIDIA 的 Nemotron 3 Content Safety，以及覆盖多语言的 X-Guard，定义了 2026 年的安全分类器技术栈。garak、PyRIT、NVIDIA Aegis 和 promptfoo 成为标准对抗性评测工具。NeMo Guardrails v0.12 将它们串联成生产管线。本毕设将这一切组装在一起：围绕目标应用的分层安全框架、运行 6+ 攻击家族的自主红队 Agent，以及产出可衡量无害性增量的宪法自批评运行。

**类型：** 毕业设计
**语言：** Python（安全管线、红队）、YAML（策略配置）
**前置课程：** Phase 10（从零构建 LLM）、Phase 11（LLM 工程）、Phase 13（工具）、Phase 14（Agent 工程）、Phase 18（伦理、安全与对齐）
**涵盖阶段：** P10 · P11 · P13 · P14 · P18
**时间：** ~25 小时

## 问题描述

2026 年 LLM 安全的前沿不在于分类器是否有效（大致有效），而在于如何在生产应用周围正确组合它们，既不过度拒绝也不留下明显漏洞。Llama Guard 4 处理英文策略违规。X-Guard（132 种语言）处理多语言越狱。ShieldGemma-2 捕获基于图像的提示注入。NVIDIA Nemotron 3 Content Safety 覆盖企业类别。Anthropic 的 Constitutional Classifiers 是训练时使用而非服务时的独立方法。

攻击演进同样重要。PAIR 和 TAP 自动化越狱发现。GCG 运行基于梯度的后缀攻击。多轮和代码切换攻击利用 Agent 记忆。任何部署的 LLM 都需要一个红队靶场——garak 和 PyRIT 是标准驱动器——加上有记录的缓解措施和 CVSS 评分的发现。

你将加固一个目标应用（一个 8B 指令微调模型或其他毕设的 RAG 聊天机器人），对它运行 6+ 攻击家族，并产出前/后无害性测量。

## 核心概念

安全管线有五层。**输入净化**：剥离零宽字符、解码 base64/rot13、Unicode 归一化。**策略层**：NeMo Guardrails v0.12 护栏（域外、毒性、PII 提取）。**分类器门**：输入端 Llama Guard 4、非英文 X-Guard、图像输入 ShieldGemma-2。**模型**：目标 LLM。**输出过滤**：输出端 Llama Guard 4、Presidio PII 清洗、引用强制（适用时）。**HITL 层**：标记为高风险的输出进入 Slack 队列。

红队靶场在调度器上运行。PAIR 和 TAP 自主发现越狱。GCG 运行基于梯度的后缀攻击。ASCII / base64 / rot13 编码攻击。多轮攻击（人设采用、记忆利用）。代码切换攻击（英语混搭斯瓦希里语或泰语）。每次运行产出结构化发现文件，含 CVSS 评分和披露时间线。

宪法自批评运行是训练时干预。取 1k 个有害尝试提示，让模型草拟回复，按书面宪法（不伤害规则）进行批评，并在批评循环上重新训练。在保留评测集上衡量前/后无害性增量。

## 架构

```
请求（文本 / 图像 / 多语言）
      │
      ↓
输入净化（剥离零宽字符、解码、归一化）
      │
      ↓
NeMo Guardrails v0.12 护栏（域外、策略）
      │
      ↓
分类器门：
  Llama Guard 4（英文）
  X-Guard（多语言，132 种语言）
  ShieldGemma-2（图像提示）
  Nemotron 3 Content Safety（企业）
      │
      ↓ （允许）
目标 LLM
      │
      ↓
输出过滤：Llama Guard 4 + Presidio PII + 引用检查
      │
      ↓
标记输出的 HITL 层

并行：
  红队调度器
    -> garak（经典攻击）
    -> PyRIT（编排式红队）
    -> 自主越狱 Agent（PAIR + TAP）
    -> GCG 后缀攻击
    -> 多语言 / 代码切换
    -> 多轮人设采用

输出：CVSS 评分发现 + 披露时间线 + 前/后无害性增量
```

## 技术栈

- 安全分类器：Llama Guard 4、ShieldGemma-2、NVIDIA Nemotron 3 Content Safety、X-Guard
- 护栏框架：NeMo Guardrails v0.12 + OPA
- 红队驱动器：garak（NVIDIA）、PyRIT（Microsoft Azure）、NVIDIA Aegis、promptfoo
- 越狱 Agent：PAIR（Chao et al., 2023）、Tree-of-Attacks（TAP）、GCG 后缀
- 宪法训练：Anthropic 风格自批评循环 + 批评上的 SFT
- PII 清洗：Presidio
- 目标：8B 指令微调模型或其他毕设的 RAG 聊天机器人

## 构建步骤

1. **目标搭建。** 在 vLLM 上部署一个 8B 指令微调模型（或复用其他毕设的 RAG 聊天机器人）。这是被测应用。

2. **安全管线封装。** 在目标周围接入五层管线。验证每层独立可观测（Langfuse 中每层一个 span）。

3. **分类器覆盖。** 加载 Llama Guard 4、X-Guard（多语言）、ShieldGemma-2（图像）。在小型标注集上分别运行以建立基线。

4. **红队调度器。** 调度 garak、PyRIT、PAIR Agent、TAP Agent、GCG 运行器、多轮攻击者和代码切换攻击者。每个在独立队列上运行。

5. **攻击套件。** 六个攻击家族：(1) PAIR 自动越狱，(2) TAP 攻击树，(3) GCG 梯度后缀，(4) ASCII / base64 / rot13 编码，(5) 多轮人设，(6) 多语言代码切换。报告每家族成功率。

6. **宪法自批评。** 策划 1k 个有害尝试提示。每个提示，目标草拟回复。批评 LLM 按书面宪法评分（"不伤害"、"引用证据"、"拒绝非法请求"）。批评反对的提示被重写；目标在批评改进对上微调。在保留评测集上衡量前/后无害性。

7. **过度拒绝测量。** 在良性提示套件（如 XSTest）上追踪误报率。目标必须在良性问题上保持有用性。

8. **CVSS 评分。** 每个成功越狱，按 CVSS 4.0 评分（攻击向量、复杂度、影响）。产出披露时间线和缓解计划。

9. **靶场自动化。** 以上所有在 cron 上运行；发现写入队列；过度拒绝回归告警发送到 Slack。

## 使用示例

```
$ safety probe --model=target --family=PAIR --budget=50
[attacker]   PAIR Agent 在目标上运行
[attack]     尝试 1/50：伪装查询为学术研究 ... 已阻止
[attack]     尝试 2/50：诉诸角色扮演 ... 已阻止
[attack]     尝试 3/50：链式思维诱导 ... 成功
[finding]    CVSS 4.8 中等：目标上的角色扮演绕过
[range]      50 次中 7 次成功（14% 成功率）
```

## 交付物

`outputs/skill-safety-harness.md` 是交付物。生产级分层安全管线加可复现红队靶场，含前/后无害性增量。

| 权重 | 评分项 | 衡量方式 |
|:-:|---|---|
| 25 | 攻击面覆盖 | 6+ 攻击家族、2+ 种语言 |
| 20 | 真阳性/假阳性权衡 | 攻击阻止率 vs XSTest 良性通过率 |
| 20 | 自批评增量 | 保留评测集上的前/后无害性 |
| 20 | 文档和披露 | CVSS 评分发现 + 时间线 |
| 15 | 自动化和可重复性 | 所有在 cron 上运行并带告警 |
| **100** | | |

## 练习

1. 对 RAG 聊天机器人运行 garak 的提示注入插件，对比有/无输出过滤层的攻击成功率。

2. 添加第七个攻击家族：通过检索文档的间接提示注入。衡量所需额外防御。

3. 实现"拒绝并提供帮助"模式：当护栏阻止时，目标提供更安全的相关回答而非生硬拒绝。衡量 XSTest 增量。

4. 多语言覆盖差距：找到 X-Guard 表现不佳的语言。提出针对该语言的微调数据集。

5. 在 30B 模型上运行宪法自批评，衡量增量是否随规模扩展。

## 关键术语

| 术语 | 业界说法 | 实际含义 |
|------|---------|---------|
| 分层安全 | "纵深防御" | 输入、门、输出、HITL 多重护栏 |
| Llama Guard 4 | "Meta 的安全分类器" | 2026 年参考级输入/输出内容分类器 |
| PAIR | "越狱 Agent" | LLM 驱动越狱发现的论文（Chao et al.） |
| TAP | "攻击树" | PAIR 的树搜索变体 |
| GCG | "贪婪坐标梯度" | 基于梯度的对抗性后缀攻击 |
| 宪法自批评 | "Anthropic 风格训练" | 目标草拟 -> 批评评分 -> 重写 -> 重训练 |
| XSTest | "良性探针集" | 过度拒绝回归基准 |
| CVSS 4.0 | "严重性评分" | 安全发现的标准漏洞评分 |

## 延伸阅读

- [Anthropic Constitutional Classifiers](https://www.anthropic.com/research/constitutional-classifiers) — 训练时参考
- [Meta Llama Guard 4](https://ai.meta.com/research/publications/llama-guard-4/) — 2026 年输入/输出分类器
- [Google ShieldGemma-2](https://huggingface.co/google/shieldgemma-2b) — 图像 + 多模态安全
- [NVIDIA Nemotron 3 Content Safety](https://developer.nvidia.com/blog/building-nvidia-nemotron-3-agents-for-reasoning-multimodal-rag-voice-and-safety/) — 企业参考
- [X-Guard（arXiv:2504.08848）](https://arxiv.org/abs/2504.08848) — 132 种语言多语言安全
- [garak](https://github.com/NVIDIA/garak) — NVIDIA 红队工具包
- [PyRIT](https://github.com/Azure/PyRIT) — 微软红队框架
- [NeMo Guardrails v0.12](https://docs.nvidia.com/nemo-guardrails/) — 护栏框架
- [PAIR（arXiv:2310.08419）](https://arxiv.org/abs/2310.08419) — 越狱 Agent 论文