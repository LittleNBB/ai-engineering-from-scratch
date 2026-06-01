# FIPA-ACL 和言语行为的遗产

> 在 MCP、A2A 之前，有 FIPA-ACL。2000 年，IEEE 智能物理代理基金会（Foundation for Intelligent Physical Agents）批准了一种智能体通信语言，包含 20 个施为语（performatives）、2 种内容语言和一套交互协议——合同网、订阅/通知、请求-条件。它因本体论（ontology）开销对 Web 来说过于沉重而从工业界淡出，但多智能体系统的 LLM 复兴正在悄悄重新实现相同的理念，只是没有形式语义：JSON 契约替代施为语，自然语言替代本体论。本课认真研读 FIPA-ACL，以便你看清 2026 年的哪些协议决策是重新发明，哪些是真正的新东西，以及当前浪潮将在哪里重新发现 2000 年代已经解决的问题。

**类型：** Learn（学习）
**语言：** Python（标准库）
**前置课程：** Phase 16 · 01（Why Multi-Agent）
**时间：** ~60 分钟

## 问题

2026 年的智能体协议（agent-protocol）版图非常繁忙：MCP 用于工具调用、A2A 用于智能体间通信、ACP 用于企业审计、ANP 用于去中心化信任、NLIP 用于自然语言内容，再加上 CA-MCP 和二十多个研究提案。每个规范都宣称自己是基础性的。

坦率地说，它们中的大多数都在重新发现一个非常具体的、二十年前的决策树。Austin（1962）和 Searle（1969）的言语行为理论（speech-act theory）告诉我们"话语即行动"。KQML（1993）将其转化为线路协议。FIPA-ACL（2000 年批准）产生了参考标准：20 个施为语、内容语言 SL0/SL1、合同网（contract-net）和订阅-通知（subscribe-notify）的交互协议。JADE 和 JACK 是 Java 参考平台。这项工作在 2010 年左右逐渐淡出，因为本体论开销太重，而 Web 赢了。

当你看到 MCP 的 `tools/call`、A2A 的任务生命周期或 CA-MCP 的共享上下文存储时，你看到的是 FIPA 决策的一个更柔和的、JSON 原生的翻版。了解这段遗产能告诉你两件事：哪些新的"创新"实际上是重新发明，以及哪些旧的失败模式会被新规范重新发现。

## 核心概念

### 言语行为（Speech Acts），一段话概括

Austin 注意到，有些句子不是在描述世界——而是在改变它。"我承诺。""我请求。""我宣布。"他把这些称为施为性话语（performative utterances）。Searle 将其形式化为五类：断言类（assertive）、指令类（directive）、承诺类（commissive）、表达类（expressive）、宣告类（declarative）。KQML（Finin 等，1993）为软件智能体将其操作化：一条消息就是一个施为语（行为）加上内容（行为的对象）。FIPA-ACL 清理了 KQML 的缺陷，并围绕 20 个施为语进行了标准化。

### 二十个 FIPA 施为语（部分列表）

| 施为语（Performative） | 意图 |
|---|---|
| `inform` | "我告诉你 P 为真" |
| `request` | "我请求你做 X" |
| `query-if` | "P 是否为真？" |
| `query-ref` | "X 的值是什么？" |
| `propose` | "我提议我们做 X" |
| `accept-proposal` | "我接受该提议" |
| `reject-proposal` | "我拒绝该提议" |
| `agree` | "我同意做 X" |
| `refuse` | "我拒绝做 X" |
| `confirm` | "我确认 P 为真" |
| `disconfirm` | "我否认 P" |
| `not-understood` | "你的消息无法解析" |
| `cfp` | "征集关于 X 的提案" |
| `subscribe` | "当 X 变化时通知我" |
| `cancel` | "取消正在进行的 X" |
| `failure` | "我尝试了 X 但失败了" |

完整列表见 `fipa00037.pdf`（FIPA ACL 消息结构）。重点不是背诵它——而是每一个施为语都对应着一个 LLM 协议最终会重新添加的原语。

### 典型的 FIPA-ACL 消息

```
(inform
  :sender       agent1@platform
  :receiver     agent2@platform
  :content      "((price IBM 83))"
  :language     SL0
  :ontology     finance
  :protocol     fipa-request
  :conversation-id   conv-42
  :reply-with   msg-17
)
```

七个字段承载协议信封；一个字段（`content`）承载有效载荷。其余字段正是你每次在 JSON 协议上拼接重试、线程和本体论时都会重新发明的东西。

### 两个遗留平台

**JADE**（Java Agent DEvelopment framework，1999–2020s）是最常用的 FIPA 兼容运行时。智能体继承一个基类，交换 ACL 消息，在容器内运行，并使用"行为"（behaviors）进行协调。交互协议库附带了合同网、订阅-通知、请求-条件和提议-接受。

**JACK**（Agent Oriented Software，商业产品）强调在 FIPA 消息之上进行 BDI（Belief-Desire-Intention，信念-愿望-意图）推理。更正式，采用更少。

两者都在 Web 技术栈吞噬多智能体用例后走向衰落。MCP 和 A2A 是 2026 年的运行时"容器"。

### FIPA 为何淡出

- **本体论开销。** FIPA 要求共享本体论来解析 `content`。就本体论达成一致是一个长达数年的标准化过程。Web 只用了 HTTP + JSON。
- **没人用的形式语义。** SL（Semantic Language，语义语言）提供了严格的真值条件，但大多数生产系统使用自由格式的内容，忽略了形式化。
- **工具锁定。** JADE 只支持 Java；JACK 是商业的。多语言团队绕开了两者。
- **互联网赢得了技术栈。** REST，然后是 JSON-RPC，然后是 gRPC，取代了 ACL 的传输层。

### LLM 复兴是 FIPA-lite

对比 FIPA `request` 和 MCP `tools/call`：

```
(request                                {
  :sender  agent1                         "jsonrpc": "2.0",
  :receiver tool-server                   "method":  "tools/call",
  :content "(lookup stock IBM)"           "params":  {"name":"lookup_stock",
  :ontology finance                                   "arguments":{"symbol":"IBM"}},
  :conversation-id c42                    "id": 42
)                                        }
```

相同的信封，不同的语法。两者都携带：谁、给谁、意图、有效载荷、关联 ID。两者都不是对对方的革命——它们是同一设计上的不同权衡。

Liu 等人 2025 年的综述（"A Survey of Agent Interoperability Protocols: MCP, ACP, A2A, ANP"，arXiv:2505.02279）明确了这一谱系：MCP 对应工具使用的言语行为，A2A 对应智能体对等的言语行为，ACP 对应审计追踪的言语行为，ANP 对应去中心化身份扩展。新规范是 ACL 的后裔，使用 JSON 语法和更宽松的语义。

### 权衡，直白地说

**FIPA 给你而现代规范放弃的：**

- 形式语义——你可以证明 `inform` 意味着发送者相信内容。
- 一个规范的施为语目录——你不必重新争论"我们是否需要一个 `cancel`？"
- 数十年的交互协议模式——合同网、订阅-通知、提议-接受——具有已知的正确性属性。

**现代规范给你而 FIPA 没有的：**

- JSON 原生的有效载荷，兼容所有现代工具。
- LLM 可以解释的自然语言内容，无需手工编码本体论。
- Web 技术栈传输（HTTP、SSE、WebSocket）。
- 通过自描述文档进行能力发现（MCP `listTools`、A2A Agent Card）。

更宽松的意图语义，更容易实现。这就是精确的权衡。

### 值得移植的交互协议

FIPA 附带了约 15 个交互协议。有三个值得带入 LLM 多智能体系统：

1. **合同网协议（Contract Net Protocol，CNP）。** 管理者发出 `cfp`（征集提案）；竞标者回复 `propose`；管理者接受/拒绝。这是经典的市场任务模式（Phase 16 · 16 Negotiation）。
2. **订阅/通知（Subscribe/Notify）。** 订阅者发送 `subscribe`；每当主题变化时发布者发送 `inform`。这就是 2026 年的每个事件总线。
3. **请求-条件（Request-When）。** "当条件 Y 满足时执行 X。"带有前置条件的延迟操作。2026 年的对应物是持久化工作流引擎中的延迟任务（Phase 16 · 22 Production Scaling）。

每一个都能干净地映射到现代消息队列、HTTP + 轮询或 SSE 流。

### 放弃本体论后会出什么问题

没有共享本体论，智能体从自然语言内容中推断含义。有据可查的 2026 年失败模式是**语义漂移（semantic drift）**：两个智能体对同一个词（`"customer"`）使用略有不同的概念，接收方智能体基于错误的解释采取行动，没有模式验证器能捕获这一点。FIPA 的本体论要求本可以在解析时拒绝该消息。

不采用完整本体论的缓解措施：

- 对 `content` 使用 JSON Schema——在线路上拒绝结构错误。
- 类型化工件（A2A）——拒绝错误的模态。
- 在信封中使用显式施为语——即使内容是自然语言，也能使意图明确。

### 2026 年规范与言语行为遗产的映射

| 现代规范 | FIPA 对应物 | 保留了什么 | 放弃了什么 |
|---|---|---|---|
| MCP `tools/call` | `request` | 显式意图、关联 ID | 形式语义、本体论 |
| MCP `resources/read` | `query-ref` | 显式意图、关联 ID | 形式语义 |
| A2A 任务生命周期 | 合同网 + 请求-条件 | 异步生命周期、状态转换 | 形式完备性保证 |
| A2A 流式事件 | 订阅/通知 | 异步推送 | 类型化谓词订阅 |
| CA-MCP 共享上下文 | 黑板（Hayes-Roth 1985） | 多写入者共享内存 | 逻辑一致性模型 |
| NLIP | 自然语言内容 | LLM 原生 | Schema |

从上到下读这张表，模式是：保留结构原语，放弃形式化，让 LLM 弥合歧义。

## 动手实现

`code/main.py` 实现了一个纯标准库的 FIPA-ACL 转换器。它编码和解码标准的 ACL 信封，并展示每个 MCP / A2A 消息形式如何归约到相同的七个字段。演示内容：

- 将 5 条 MCP 风格和 A2A 风格的消息编码为 FIPA-ACL。
- 将 FIPA-ACL 解码回现代等价物。
- 使用 `cfp`、`propose`、`accept-proposal`、`reject-proposal` 在一个管理者和三个竞标者之间运行一个玩具级合同网谈判。

运行：

```
python3 code/main.py
```

输出是一个并排跟踪，展示每条现代消息在其 2026 年 JSON 形式和 FIPA-ACL 形式下的对比，然后是合同网投标的往返。相同的协议原语在往返中存活；只有语法不同。

## 实际应用

`outputs/skill-fipa-mapper.md` 是一个技能，读取任何智能体协议规范并生成 FIPA-ACL 映射。在采用新协议之前使用它来回答："这是真正的新东西，还是 `inform` 加了 JSON 语法？"

## 交付建议

不要把 FIPA-ACL 搬回来。把它的检查清单搬回来：

- 每条消息的意图原语（施为语）是什么？
- 是否有关联 ID 用于请求-响应和取消？
- 是否有显式的内容语言（JSON-RPC、纯文本、结构化类型化工件）？
- 交互协议是一等公民，还是你在从头重新实现合同网？
- 当两个智能体对内容含义产生分歧时会发生什么（语义漂移）？

在将任何新协议投入生产之前，为它记录这五个问题。

## 练习

1. 运行 `code/main.py`。观察往返编码。识别哪些 FIPA 施为语对应 `tools/call`、`resources/read` 和 A2A 任务创建。
2. 用 `cancel` 施为语扩展合同网演示，让管理者可以在投标中途撤回任务。`cancel` 解决了哪些仅靠重试无法解决的失败场景？
3. 阅读 FIPA ACL Message Structure（http://www.fipa.org/specs/fipa00037/）第 4.1–4.3 节。选择一个本课未涵盖的施为语，描述其现代 JSON-RPC 对应物。
4. 阅读 Liu 等人的 arXiv:2505.02279。对 MCP、A2A、ACP、ANP，分别列出它们保留和放弃的 FIPA 施为语族。
5. 为你自己系统中的 `request` 施为语的 `content` 字段设计一个最小的 JSON Schema。该 Schema 给你带来了哪些纯自然语言没有的东西，又付出了什么代价？

## 关键术语

| 术语 | 常见说法 | 实际含义 |
|------|---------|---------|
| Speech act（言语行为） | "做某事的话语" | Austin/Searle：话语即行动。ACL 的理论基础。 |
| FIPA | "那个老 XML 东西" | IEEE 智能物理代理基金会。2000 年标准化了 ACL。 |
| ACL（智能体通信语言） | "Agent Communication Language" | FIPA 的信封格式：施为语 + 内容 + 元数据。 |
| Performative（施为语） | "动词" | 消息的意图类别：`inform`、`request`、`propose`、`cfp` 等。 |
| KQML | "FIPA 的前身" | Knowledge Query and Manipulation Language（1993）。更简单、更窄。 |
| Ontology（本体论） | "共享词汇表" | 内容语言所讨论概念的形式化定义。 |
| SL0 / SL1 | "FIPA 内容语言" | Semantic Language 0 级和 1 级——形式化内容语言家族。 |
| Contract Net（合同网） | "任务市场" | 管理者发出 cfp；竞标者提案；管理者接受。经典的交互协议。 |
| Interaction protocol（交互协议） | "消息模式" | 一系列具有已知正确性的施为语序列：请求-条件、订阅-通知等。 |

## 延伸阅读

- [Liu 等 — A Survey of Agent Interoperability Protocols: MCP, ACP, A2A, ANP](https://arxiv.org/html/2505.02279v1) — 2025 年权威综述，将现代规范与 FIPA 遗产联系起来
- [FIPA ACL Message Structure Specification (fipa00037)](http://www.fipa.org/specs/fipa00037/) — 2000 年批准的信封格式
- [FIPA Communicative Act Library Specification (fipa00037)](http://www.fipa.org/specs/fipa00037/) — 完整的施为语目录
- [MCP specification 2025-11-25](https://modelcontextprotocol.io/specification/2025-11-25) — `request`/`query-ref` 的现代工具使用等价物
- [A2A specification](https://a2a-protocol.org/latest/specification/) — 合同网和订阅-通知的现代智能体对等等价物