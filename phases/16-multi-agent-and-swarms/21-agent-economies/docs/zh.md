# 智能体经济、Token 激励与声誉（Agent Economies, Token Incentives, Reputation）

> 长视界自主智能体（METR 的 1 小时到 8 小时工作曲线）需要经济代理能力。新兴的**五层架构**是：**DePIN**（物理计算）→ **身份**（W3C DID + 声誉资本）→ **认知**（RAG + MCP）→ **结算**（账户抽象）→ **治理**（Agentic DAOs）。生产级智能体激励网络包括 **Bittensor**（TAO 子网奖励特定任务模型）、**Fetch.ai / ASI Alliance**（ASI-1 Mini LLM + FET token）和 **Gonka**（基于 Transformer 的 PoW，将计算重新分配给有生产力的 AI 任务）。学术工作：AAMAS 2025 的去中心化 LaMAS 使用 **Shapley 值归因**来公平奖励贡献智能体；Google Research 的"Mechanism design for large language models"提出了在单调聚合下的**token 拍卖**和二价支付。本课构建一个最小智能体市场，将 Shapley 值归因应用于多智能体流水线，并运行二价 token 拍卖，使博弈论机制具体落地。

**类型：** Learn（学习）
**语言：** Python（标准库）
**前置课程：** Phase 16 · 16（Negotiation and Bargaining），Phase 16 · 09（Parallel Swarm Networks）
**时间：** ~75 分钟

## 问题

当智能体联合产出价值但需要单独获得奖励时，多智能体系统变得复杂。经典机制——平均分配、最后贡献者全得——不公平或可被博弈。基于联盟的 Shapley 值奖励在构造上是公平的，但计算昂贵。2025-2026 年的文献推动了有用的近似：Shapley 采样、单调聚合拍卖，以及从确认贡献中积累的链上声誉。

在信用归属之外，该领域已转向实际的经济智能体：Bittensor TAO 奖励挖矿计算来微调子网特定模型，Fetch.ai/ASI 用 FET token 奖励 ASI-1 Mini LLM 的使用，Gonka 将 Transformer 工作量证明重新分配给有生产力的 AI 任务。自主交易的智能体现在就存在；问题是如何对齐激励。

本课将智能体经济视为一个特定的问题族——信用归属、机制设计和声誉——用最小的数学构建每个，使概念扎根。

## 核心概念

### 五层智能体经济架构

1. **DePIN（物理计算）。** 租用 GPU、存储、带宽的去中心化基础设施。Bittensor 子网、Render Network、Akash。不是智能体特定的；智能体使用它。
2. **身份。** W3C 去中心化标识符（DID）给每个智能体一个独立于任何平台的持久 ID。声誉积累到 DID 上。智能体网络协议（ANP）使用 DID 作为发现层。
3. **认知。** 智能体的推理循环：LLM + RAG + MCP。这是其他阶段构建的内容。
4. **结算。** 账户抽象（ERC-4337）让智能体从自己的余额支付 gas 而无需持有 ETH。智能体可以为服务、互相之间或计算付费。
5. **治理。** Agentic DAOs：人类*和*智能体对协议变更投票的治理结构，投票权与声誉挂钩。

不是每个生产系统都使用全部五层。Bittensor 使用 1、2、部分 3、部分 4、不使用 5。OpenAI 智能体除了 3 之外都不使用。该架构是参考地图，不是要求。

### Bittensor、Fetch.ai、Gonka —— 运行中的系统

**Bittensor（TAO）。** 子网是专门任务（语言建模、图像生成、预测）。矿工提交模型输出。验证者排名；加权评分分配 TAO 奖励。每个子网有自己的评估。经济学教训：为特定任务的输出质量付费，而非使用的计算。

**Fetch.ai / ASI Alliance。** ASI-1 Mini LLM 运行在 Fetch.ai 网络上；用户用 FET token 支付推理。智能体作为对等方的叙事在这里更强：Fetch 上的智能体可以调用另一个智能体执行任务并用 FET 支付。

**Gonka。** Transformer 工作量证明："工作"是 Transformer 的前向传播。矿工通过运行具有已知正确输出（来自训练数据）的推理任务来赚取。资源生产性 PoW 而非基于哈希的 PoW。

截至 2026 年 4 月，三者都是生产级的。收益分配不同。Bittensor 奖励相对于子网验证者的质量；Fetch 奖励由付费用户衡量的效用；Gonka 奖励可验证的推理工作。

### Shapley 值信用归属

三个智能体协作完成任务。输出得分 0.8。谁贡献了什么？

Shapley 值：满足四个公理（效率、对称性、线性、零性）的唯一信用分配。对于智能体 `i`：

```
shapley(i) = (1/N!) * 对所有排列 O 求和 (v(S_i_O ∪ {i}) - v(S_i_O))
```

其中 `S_i_O` 是排列 `O` 中 `i` 之前的智能体集合。实践中：枚举所有排列，记录每个排列中每个智能体的边际贡献，取平均。

对于 N=3 个智能体，有 6 个排列。对于 N=10，360 万——所以实践中你采样排列而非枚举。

### 二价拍卖用于聚合

Google Research（"Mechanism design for large language models"）提出了用于聚合 LLM 输出的二价 token 拍卖。设置：N 个智能体各提出一个完成；每个对被选中有一个私人价值。拍卖师选择最高价值的提案并支付*第二高的*价值。在单调聚合下（价值取决于选择了哪个提案，而非投了多少标），这是诚实的——智能体报出其真实价值。

为什么这对 LLM 系统重要：你可以将完成任务外包给具有不同定价的多个智能体；拍卖选出最佳 + 公平支付，智能体没有误报的激励。

### 声誉资本

DID 绑定的声誉分数从确认的贡献中积累。一个简单的更新规则：

```
rep(i, t+1) = alpha * rep(i, t) + (1 - alpha) * contribution_quality(i, t)
```

衰减因子 `alpha` 接近 1。声誉：

- 对路由决策来说读取便宜（"将困难任务发送给高声誉智能体"）。
- 伪造昂贵（随时间积累，绑定到 DID）。
- 可以被削减：未通过验证的贡献会被扣除。

### AAMAS 2025 去中心化 LaMAS

LaMAS 提案（AAMAS 2025）结合了：DID 身份、Shapley 值信用归属和简单的拍卖机制。关键主张：去中心化信用归属步骤使系统可审计且免疫单点操纵。

### 经济学何时崩溃

- **价格预言机操控。** 如果信用函数可以被博弈，智能体就会博弈它。每个机制都需要对抗性测试。
- **女巫攻击（Sybil attacks）。** 一个操作者启动 N 个假智能体来膨胀自己的贡献。DID 减缓但不能阻止；声誉伪造成本是缓解措施。
- **验证成本。** 信用归属只和验证者一样公平。如果验证便宜（小型 LLM），它可被博弈；如果昂贵（人类小组），系统不能扩展。
- **监管悬置。** 智能体经济与金融监管交叉。截至 2026 年，Bittensor、Fetch 和 Gonka 在某些司法管辖区都处于法律灰色地带。

### 智能体经济适用场景

- **具有异质操作者的开放网络。** 没有单一团队控制所有智能体。
- **可验证输出。** 没有验证，信用归属就是猜测。
- **长视界工作流。** 一次性任务不受益于声誉积累。
- **代币化支付在你的司法管辖区合法。**

在封闭的企业系统中，经济学让位于更简单的分配（管理者分配工作，指标是内部的）。经济学文献主要适用于开放网络。

## 动手实现

`code/main.py` 实现了：

- `shapley(value_fn, agents)` — 通过枚举对小 N 进行精确 Shapley 计算。
- `second_price_auction(bids)` — 诚实机制；赢家支付第二高价。
- `Reputation` — DID 绑定的声誉，带指数衰减和削减。
- 演示 1：三个智能体协作，精确 Shapley 分配信用。
- 演示 2：五个智能体竞标任务槽；二价拍卖选择赢家 + 支付。
- 演示 3：100 轮任务分配给具有异质声誉的智能体；声誉加权路由击败随机。

运行：

```
python3 code/main.py
```

预期输出：每个智能体的 Shapley 值；展示诚实投标均衡的拍卖结果；声誉加权路由在预热后比随机高出 10-20% 的质量增益。

## 实际应用

`outputs/skill-economy-designer.md` 设计一个最小智能体经济：身份层选择、信用归属机制、支付机制、声誉规则。

## 交付建议

2026 年运行智能体经济：

- **从声誉开始，而非代币。** 声誉实现便宜且单独就有价值；代币增加法律和经济复杂性。
- **先验证再奖励。** 没有独立验证步骤绝不分配信用。自报告质量积累女巫博弈。
- **Shapley 采样，而非 Shapley 精确。** 采样 100-1000 个排列；精确枚举不扩展。
- **限制衰减因子和声誉下限。** 无界衰减抹去合法贡献者；太慢的衰减奖励过时的高声誉智能体。
- **对抗性审计机制。** 在开放网络之前运行红队场景。每个机制都有博弈论；你想找到漏洞，而非攻击者。

## 练习

1. 运行 `code/main.py`。确认 Shapley 值之和等于总值（效率公理）。改变价值函数；Shapley 分配是否按预期方向变化？
2. 实现 Shapley *采样*（对 K 个排列的蒙特卡洛）。K 如何影响近似精度？与 N=4 的精确值比较。
3. 在拍卖前实现联盟形成步骤：智能体可以合并成团队并作为一个单位竞标。哪些联盟形成？结果是否比个体竞标帕累托更优？
4. 阅读 Google Research 的机制设计博文。识别一个如果违反会破坏诚实性的假设。该失败模式在 LLM 设置中是什么样的？
5. 阅读 AAMAS 2025 去中心化 LaMAS 论文。在合成任务上对 10 个智能体实现他们的 Shapley 步骤。精确计算需要多长时间？100 次采样能多接近？

## 关键术语

| 术语 | 常见说法 | 实际含义 |
|------|---------|---------|
| DePIN | "去中心化物理基础设施" | 代币激励的计算/存储/带宽。Bittensor、Akash、Render。 |
| DID（去中心化标识符） | "去中心化 ID" | W3C 规范的可移植 ID。智能体声誉绑定到 DID，而非平台。 |
| ERC-4337 | "账户抽象" | 可赞助 gas 的合约账户，使智能体支付成为可能。 |
| Shapley 值 | "公平信用归属" | 满足效率、对称性、线性、零性的唯一分配。 |
| Second-price auction（二价拍卖） | "Vickrey 拍卖" | 诚实机制：赢家支付第二高出价。兼容单调聚合。 |
| Reputation capital（声誉资本） | "积累的质量分数" | 来自确认贡献的 DID 绑定分数；随时间衰减。 |
| Agentic DAO | "智能体 + 人类治理" | 以智能体投票者为一等公民的 DAO，投票权与声誉挂钩。 |
| TAO / FET / GPU 积分 | "代币面额" | Bittensor TAO、Fetch.ai FET、各种 DePIN 代币。 |

## 延伸阅读

- [The Agent Economy](https://arxiv.org/abs/2602.14219) — 2026 年五层智能体经济架构综述
- [Google Research — Mechanism design for large language models](https://research.google/blog/mechanism-design-for-large-language-models/) — 带单调聚合的 token 拍卖
- [AAMAS 2025 — decentralized LaMAS](https://www.ifaamas.org/Proceedings/aamas2025/pdfs/p2896.pdf) — Shapley 值信用归属
- [Bittensor TAO documentation](https://docs.bittensor.com/) — 子网结构和奖励分配
- [Fetch.ai / ASI Alliance](https://fetch.ai/) — ASI-1 Mini LLM 和 FET token
- [W3C Decentralized Identifiers (DIDs) spec](https://www.w3.org/TR/did-core/) — 身份基础