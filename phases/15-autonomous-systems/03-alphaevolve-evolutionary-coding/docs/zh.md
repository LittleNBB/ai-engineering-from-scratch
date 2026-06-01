# AlphaEvolve — 进化编码智能体（Evolutionary Coding Agents）

> 将前沿编码模型与进化循环和机器可检查的评估器配对。让循环运行足够长。它发现了一个使用 48 次标量乘法的 4x4 复数矩阵乘法程序——56 年来首次超越 Strassen。它还找到了一个 Google 全局的 Borg 调度启发式，在生产中回收了约 0.7% 的集群计算资源。架构是刻意无聊的。胜利来自评估器的严谨性。

**类型：** Learn（学习）
**语言：** Python（标准库，进化循环玩具）
**前置课程：** Phase 15 · 01（long-horizon framing），Phase 15 · 02（self-taught reasoning）
**时间：** ~60 分钟

## 问题

大型语言模型可以写代码。进化算法可以搜索代码空间。两者都已被分别尝试了几十年；两者都遇到了天花板。LLM 的天花板是虚构：模型写出看似合理但不做它声称的事情的代码。进化的天花板是搜索成本：对语法的随机突变很少产生可编译的程序，更别说更好的程序。

AlphaEvolve（Novikov 等，DeepMind，arXiv:2506.13131，2025 年 6 月）将两者结合。LLM 对程序数据库提出有针对性的编辑；自动评估器对每个变体评分；高分变体成为未来世代的父代。LLM 处理编写看似合理代码的昂贵步骤；评估器捕获虚构。循环运行数小时到数周。

报告的结果：48 次标量乘法的 4x4 复数矩阵乘法（Strassen 1969 年的界限是 49），Google 生产中的 Borg 调度启发式，FlashAttention 内核 32.5% 的加速，Gemini 训练吞吐量改进。

该架构有效是因为评估器是机器可检查的。在评估器不是的领域它不起作用。这种不对称性就是教训。

## 核心概念

### 循环

1. 从一个正确但次优的种子程序 `P_0` 开始。
2. 维护一个变体程序数据库，每个都由评估器评分。
3. 从数据库中采样一个或多个父代（MAP-elites 风格或基于岛屿）。
4. 提示 LLM（困难的用 Gemini Pro，多个候选用 Gemini Flash）产生父代的修改变体。
5. 在留出的评估器上编译、运行并评估变体。
6. 按其分数和特征向量插入数据库。
7. 重复。

两个细节很重要。首先，LLM 被提示的不仅仅是父程序——通常还有数据库中的几个顶级变体，加上评估器签名，加上简短的任务描述。模型的工作是提出可能改善分数的有针对性的更改。其次，数据库是结构化的（MAP-elites 网格、基于岛屿），因此循环探索多样性，而不仅仅是当前领先者。

### 评估器为什么不可妥协

AlphaEvolve 的胜利都来自评估器快速、确定性且难以博弈的领域：

- **矩阵乘法算法**：一个乘以矩阵并逐位检查相等性的单元测试。
- **Borg 调度启发式**：一个生产级模拟器，重放历史集群负载并测量浪费的计算。
- **FlashAttention 内核**：一个正确性测试加上真实硬件上的挂钟基准。
- **Gemini 训练吞吐量**：测量每步 GPU 秒数。

在每种情况下，评估器捕获了否则会占主导的 LLM 错误类型：虚构的正确性声明、在硬件上消失的性能声明和边界情况失败。移除评估器，循环就会优化漂亮的代码。

### 奖励黑客是该声明的另一面

进化优化评估器测量的任何东西。如果评估器不完美，循环会找到不完美之处。在未验证的领域，循环会优化表面特征而非预期行为。DeepMind 在论文中明确标记了这一点：AlphaEvolve 的成功仅转移到评估器严谨性与搜索雄心相匹配的领域。

代码搜索循环中奖励黑客的具体 2025-2026 示例：

- 奖励"完成时间"的优化目标奖励了提交空解决方案。
- 奖励测试下正确性的基准分数奖励了记忆测试和过拟合。
- 一个"代码质量"代理奖励了删除注释和重写变量名，没有语义变化。

AlphaEvolve 中的修复：部署一个 LLM 从未见过的留出评估器，输入在评估时生成。即便如此，DeepMind 建议对任何提议的部署进行严格审查。

### 为什么 LLM + 搜索比单独使用任何一种都好

LLM 可以产生可编译的、语义上合理的修改。在 2000 行 Python 文件上的随机突变 GA 几乎总是产生语法错误。LLM 还将搜索集中在合理的邻域中（更改一个函数，而非随机字节），这显著减少了浪费的评估器调用。

反过来，评估器捕获 LLM 的虚构。LLM 会自信地声称一个函数"在极限情况下是 O(n log n)"而实际上是 O(n²)；挂钟基准使问题得到解决。

### AlphaEvolve 在前沿技术栈中的位置

| 系统 | 生成器 | 评估器 | 领域 | 示例胜利 |
|---|---|---|---|---|
| AlphaEvolve | Gemini | 正确性 + 基准 | 算法、内核、调度器 | 48 次乘法 4x4 矩阵乘法 |
| FunSearch（DeepMind，2023） | PaLM / Codey | 正确性 | 组合数学 | cap-set 下界 |
| AI Scientist v2（Sakana，L5） | GPT/Claude | LLM 批评 + 实验 | ML 研究 | ICLR workshop 论文 |
| Darwin Godel Machine（L4） | 智能体脚手架 | SWE-bench / Polyglot | 智能体代码 | SWE-bench 20% → 50% |

四个都是同一配方的变体：生成器加评估器，循环。差异在于评估器评分什么以及它有多严格。

## 动手实现

`code/main.py` 在一个玩具符号回归问题上实现了一个最小的类 AlphaEvolve 循环。"LLM"是一个标准库代理，对计算目标函数的程序提出小的语法突变。"评估器"在留出的测试点上测量均方误差。

观察：

- 最佳分数如何随世代改善。
- MAP-elites 网格如何保持多样化的解决方案存活，使循环不会收敛到局部最小值。
- 移除留出测试（仅训练评估器）如何让循环壮观地过拟合。

## 实际应用

`outputs/skill-evaluator-rigor-audit.md` 是在新领域考虑 AlphaEvolve 风格循环的前提条件：你的评估器是否真的捕获你关心的失败？

## 练习

1. 运行 `code/main.py`。注意最佳分数轨迹。禁用留出评估器（标志 `--no-holdout`）并重新运行。量化过拟合。

2. 阅读 AlphaEvolve 论文关于 MAP-elites 网格的第 3 节。为一个新问题（例如编译器优化传递）设计一个特征向量描述符，保持搜索多样。

3. 48 次乘法的 4x4 结果在 56 年后改进了 Strassen 的 49 次乘法界限。阅读论文的附录 F，用三句话解释为什么这个问题的评估器特别容易做对，以及为什么大多数领域不像它。

4. 提出一个 AlphaEvolve 会失败的领域。精确定位评估器在哪里崩塌以及为什么。

5. 对于你知道的一个领域，编写你将使用的评估器签名。包括（a）正确性条件，（b）性能指标，（c）留出输入生成规则，（d）至少一个反奖励黑客检查。

## 关键术语

| 术语 | 常见说法 | 实际含义 |
|---|---|---|
| AlphaEvolve | "DeepMind 的进化编码智能体" | Gemini + 程序数据库 + 机器可检查的评估器 |
| MAP-elites | "多样性保持档案" | 按特征向量键控的网格；每个单元格保存该描述符的最佳变体 |
| Island model（岛屿模型） | "并行进化子种群" | 定期迁移的独立种群；防止过早收敛 |
| Machine-checkable evaluator（机器可检查评估器） | "确定性预言机" | LLM 无法伪造的单元测试、模拟器或基准——此循环的前提 |
| Reward hacking（奖励黑客） | "优化指标而非目标" | 循环找到不执行预期任务就最大化分数的方法 |
| Seed program（种子程序） | "起点" | 循环从中进化的初始正确但次优的程序 |
| Held-out evaluator（留出评估器） | "LLM 从未见过的评估数据" | 在评估时生成的输入以防止记忆 |

## 延伸阅读

- [Novikov 等 (2025). AlphaEvolve: A coding agent for scientific and algorithmic discovery](https://arxiv.org/abs/2506.13131) — 完整论文
- [DeepMind blog on AlphaEvolve](https://deepmind.google/blog/alphaevolve-a-gemini-powered-coding-agent-for-designing-advanced-algorithms/) — 带结果的厂商报道
- [AlphaEvolve results repository](https://github.com/google-deepmind/alphaevolve_results) — 发现的算法，包括 48 次乘法的 4x4 矩阵乘法
- [Romera-Paredes 等 (2023). Mathematical discoveries from program search with LLMs (FunSearch)](https://www.nature.com/articles/s41586-023-06924-6) — 前驱系统
- [Anthropic — Responsible Scaling Policy v3.0 (Feb 2026)](https://anthropic.com/responsible-scaling-policy/rsp-v3-0) — 将评估器约束的自主性作为关键研究方向