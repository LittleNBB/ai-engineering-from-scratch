# 为什么需要多智能体（Multi-Agent）？

> 一个智能体撞墙了。聪明的做法不是造一个更大的智能体——而是造更多智能体。

**类型：** Learn（学习）
**语言：** TypeScript
**前置课程：** Phase 14（Agent Engineering）
**时间：** ~60 分钟

## 学习目标

- 识别单智能体天花板（上下文溢出、混合专业能力、串行瓶颈），并解释何时应该拆分为多个智能体
- 比较编排模式（Pipeline（流水线）、Parallel fan-out（并行扇出）、Supervisor（监督者）、Hierarchical（分层）），并为给定任务结构选择合适的模式
- 设计具有清晰角色边界、共享状态和通信契约的多智能体系统
- 分析多智能体复杂性（延迟、成本、调试难度）与单智能体简洁性之间的权衡

## 问题所在

你在 Phase 14 中构建了一个单智能体。它能工作——可以读取文件、运行命令、调用 API 并对结果进行推理。然后你把它指向一个真实的代码库：200 个文件、三种语言、依赖基础设施的测试，以及需要在编写代码之前研究外部 API 的需求。

智能体卡住了。不是因为 LLM 笨，而是因为任务超出了单个智能体循环能处理的范围。上下文窗口被文件内容填满。智能体忘记了 40 次工具调用之前读过的内容。它试图同时充当研究员、编码者和审阅者，结果三件事都做不好。

这就是**单智能体天花板**。每当任务需要以下条件时，你就会遇到它：

- **超过一个窗口能容纳的上下文** —— 读取 50 个文件会超过 20 万 token
- **不同阶段需要不同专业能力** —— 研究需要与代码生成不同的提示词
- **可以并行执行的工作** —— 为什么要串行读取三个文件，而不是同时读取？

## 核心概念

### 单智能体天花板

单智能体就是一个循环、一个上下文窗口、一条系统提示词。想象一下：

```
┌─────────────────────────────────────────┐
│            单智能体                      │
│                                         │
│  ┌───────────────────────────────────┐  │
│  │         上下文窗口                │  │
│  │                                   │  │
│  │  研究笔记                         │  │
│  │  + 代码文件                       │  │
│  │  + 测试输出                       │  │
│  │  + 审阅反馈                       │  │
│  │  + API 文档                       │  │
│  │  + ...                            │  │
│  │                                   │  │
│  │  ██████████████████████ 已满 ███  │  │
│  │                                   │  │
│  └───────────────────────────────────┘  │
│                                         │
│  一条系统提示词试图涵盖                  │
│  研究 + 编码 + 审阅 + 测试              │
│                                         │
│  结果：样样通，样样松                    │
└─────────────────────────────────────────┘
```

三个地方会出问题：

1. **上下文饱和** —— 工具结果不断堆积。到第 30 轮时，智能体已经消耗了 15 万 token 的文件内容、命令输出和先前推理。第 5 轮的关键细节被遗忘了。

2. **角色混乱** —— 一条说"你是研究员、编码者、审阅者和测试者"的系统提示词，会造就一个半吊子研究、半吊子编码、永远完不成审阅的智能体。

3. **串行瓶颈** —— 智能体先读文件 A，再读文件 B，再读文件 C。三次串行 LLM 调用，三次串行工具执行，没有并行。

### 多智能体解决方案

拆分工作。给每个智能体一个任务、一个上下文窗口、一条针对该任务调优的系统提示词：

```
┌──────────────────────────────────────────────────────────┐
│                    编排器（Orchestrator）                   │
│                                                          │
│  "构建一个用户管理的 REST API"                              │
│                                                          │
│         ┌──────────┬──────────┬──────────┐               │
│         │          │          │          │               │
│         ▼          ▼          ▼          ▼               │
│   ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │
│   │ 研究员   │ │  编码者  │ │  审阅者  │ │  测试者  │  │
│   │          │ │          │ │          │ │          │  │
│   │ 阅读     │ │ 根据     │ │ 检查     │ │ 运行     │  │
│   │ 文档，   │ │ 研究     │ │ 代码     │ │ 测试，   │  │
│   │ 发现     │ │ 和规格   │ │ 质量，   │ │ 报告     │  │
│   │ 模式     │ │ 编写代码 │ │ 发现缺陷 │ │ 结果     │  │
│   └─────┬────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘  │
│         │           │            │             │         │
│         └───────────┴────────────┴─────────────┘         │
│                          │                               │
│                     合并结果                               │
└──────────────────────────────────────────────────────────┘
```

每个智能体拥有：
- 专注的系统提示词（"你是一名代码审阅者。你唯一的任务就是发现缺陷。"）
- 自己的上下文窗口（不被其他智能体的工作污染）
- 清晰的输入/输出契约（接收研究笔记，输出代码）

### 真实系统中的应用

**Claude Code 子智能体** —— 当 Claude Code 使用 `Task` 生成子智能体时，它会创建一个具有作用域任务的子智能体。父智能体保持上下文干净。子智能体执行专注的工作并返回摘要。

**Devin** —— 运行一个规划智能体、一个编码智能体和一个浏览器智能体。规划智能体将工作分解为步骤。编码智能体编写代码。浏览器智能体研究文档。各自拥有独立的上下文。

**多智能体编码团队（SWE-bench）** —— SWE-bench 上表现最好的系统使用一个研究员读取代码库、一个规划者设计修复方案、一个编码者实现修复。单智能体系统得分更低。

**ChatGPT Deep Research** —— 并行生成多个搜索智能体，每个探索不同的角度，然后综合结果。

### 谱系

多智能体不是非此即彼。它是一个谱系：

```
简单 ──────────────────────────────────────────── 复杂

 单智能体      子智能体       流水线        团队         群体

 ┌───┐       ┌───┐        ┌───┐───┐    ┌───┐───┐    ┌─┐┌─┐┌─┐
 │ A │       │ A │        │ A │ B │    │ A │ B │    │ ││ ││ │
 └───┘       └─┬─┘        └───┘─┬─┘    └─┬─┘─┬─┘    └┬┘└┬┘└┬┘
               │                │        │   │       ┌┴──┴──┴┐
             ┌─┴─┐          ┌───┘───┐    │   │       │共享   │
             │ a │          │ C │ D │  ┌─┴───┴─┐    │状态   │
             └───┘          └───┘───┘  │ 消息   │    └───────┘
                                       │ 总线   │
 1 个循环    父任务 +       按阶段      │       │    N 个对等节点，
 1 个上下文  子任务        流转        └───────┘    涌现行为
                                       显式角色
```

**单智能体** —— 一个循环，一条提示词。适合简单任务。

**子智能体** —— 父智能体生成子智能体执行专注的子任务。父智能体维护计划。子智能体汇报结果。Claude Code 就是这样做的。

**流水线（Pipeline）** —— 智能体按顺序运行。智能体 A 的输出成为智能体 B 的输入。适合分阶段工作流：研究 -> 编码 -> 审阅 -> 测试。

**团队（Team）** —— 智能体通过共享消息总线并行运行。各自有角色。编排器协调。适合同时需要不同技能的场景。

**群体（Swarm）** —— 许多相同或近似的智能体，共享状态。没有固定的编排器。智能体从队列中获取工作。适合高吞吐量的并行任务。

### 四种多智能体模式

#### 模式 1：流水线（Pipeline）

```
输入 ──▶ 智能体 A ──▶ 智能体 B ──▶ 智能体 C ──▶ 输出
          (研究)      (编码)      (审阅)
```

每个智能体转换数据并向前传递。易于理解。某个阶段的失败会阻塞其余阶段。

#### 模式 2：扇出/扇入（Fan-out / Fan-in）

```
                 ┌──▶ 智能体 A ──┐
                 │               │
输入 ──▶ 拆分 ├──▶ 智能体 B ──├──▶ 合并 ──▶ 输出
                 │               │
                 └──▶ 智能体 C ──┘
```

将工作拆分给并行智能体，然后合并结果。适合可分解为独立子任务的任务。

#### 模式 3：编排者-工人（Orchestrator-Worker）

```
                     ┌──────────┐
                     │  编排者  │
                     └──┬───┬───┘
                   任务 │   │ 任务
                  ┌─────┘   └─────┐
                  ▼               ▼
            ┌──────────┐   ┌──────────┐
            │ 工人 A   │   │ 工人 B   │
            └──────────┘   └──────────┘
```

一个智能的编排者决定做什么，分配给工人，并综合结果。编排者本身就是一个拥有生成工人工具的智能体。

#### 模式 4：对等群体（Peer Swarm）

```
         ┌───┐ ◄──── 消息 ────▶ ┌───┐
         │ A │                   │ B │
         └─┬─┘                   └─┬─┘
           │                       │
      消息 │    ┌───────────┐      │ 消息
           └───▶│  共享     │◄─────┘
                │  状态     │
           ┌───▶│  / 队列   │◄─────┐
           │    └───────────┘      │
      消息 │                       │ 消息
         ┌─┴─┐                   ┌─┴─┐
         │ C │ ◄──── 消息 ────▶ │ D │
         └───┘                   └───┘
```

没有中央编排者。智能体点对点通信。决策从交互中涌现。调试更难，但可以扩展到很多智能体。

### 何时不该使用多智能体

多智能体会增加复杂性。智能体之间的每条消息都是潜在的故障点。调试从"读一个对话"变成"追踪五个智能体之间的消息"。

**保持单智能体的场景：**
- 任务能放进一个上下文窗口（工作数据不超过约 10 万 token）
- 不同阶段不需要不同的系统提示词
- 串行执行已经够快
- 任务足够简单，拆分带来的开销超过收益

**复杂性成本：**
- 每个智能体边界都是一个有损压缩步骤：智能体 A 的完整上下文被摘要为给智能体 B 的消息
- 协调逻辑（谁做什么、什么时候做、按什么顺序）本身就是 bug 来源
- 延迟增加：N 个智能体意味着最少 N 次串行 LLM 调用，如果需要来回交流则更多
- 成本翻倍：每个智能体独立消耗 token

**经验法则：** 如果一个任务少于 20 次工具调用且不超过 10 万 token，就保持单智能体。

## 动手实现

### 步骤 1：过载的单智能体

下面是一个试图做所有事情的单智能体。它有一个庞大的系统提示词和一个同时存放研究、代码和审阅结果的上下文窗口：

```typescript
type AgentResult = {
  content: string;
  tokensUsed: number;
  toolCalls: number;
};

async function singleAgentApproach(task: string): Promise<AgentResult> {
  // 单条系统提示词试图涵盖所有角色
  const systemPrompt = `You are a full-stack developer. You must:
1. Research the requirements
2. Write the code
3. Review the code for bugs
4. Write tests
Do ALL of these in a single conversation.`;

  const contextWindow: string[] = [];
  let totalTokens = 0;
  let totalToolCalls = 0;

  // 第一步：研究需求
  const research = await fakeLLMCall(systemPrompt, `Research: ${task}`);
  contextWindow.push(research.output);
  totalTokens += research.tokens;
  totalToolCalls += research.calls;

  // 第二步：编写代码——上下文已包含研究内容
  const code = await fakeLLMCall(
    systemPrompt,
    `Given this research:\n${contextWindow.join("\n")}\n\nNow write code for: ${task}`
  );
  contextWindow.push(code.output);
  totalTokens += code.tokens;
  totalToolCalls += code.calls;

  // 第三步：审阅代码——上下文已膨胀
  const review = await fakeLLMCall(
    systemPrompt,
    `Given all previous context:\n${contextWindow.join("\n")}\n\nReview the code.`
  );
  contextWindow.push(review.output);
  totalTokens += review.tokens;
  totalToolCalls += review.calls;

  return {
    content: contextWindow.join("\n---\n"),
    tokensUsed: totalTokens,
    toolCalls: totalToolCalls,
  };
}
```

这种方法的问题：
- 上下文窗口随每个阶段增长。到审阅步骤时，它已包含研究笔记、代码和先前推理。
- 系统提示词是通用的。无法为每个阶段调优。
- 没有任何并行。

### 步骤 2：专家智能体

现在拆分它。每个智能体只承担一个任务：

```typescript
type SpecialistAgent = {
  name: string;
  systemPrompt: string;
  run: (input: string) => Promise<AgentResult>;
};

// 创建专家智能体工厂函数
function createSpecialist(name: string, systemPrompt: string): SpecialistAgent {
  return {
    name,
    systemPrompt,
    run: async (input: string) => {
      const result = await fakeLLMCall(systemPrompt, input);
      return {
        content: result.output,
        tokensUsed: result.tokens,
        toolCalls: result.calls,
      };
    },
  };
}

// 研究员：只负责阅读文档和发现模式
const researcher = createSpecialist(
  "researcher",
  "You are a technical researcher. Read documentation, find patterns, and summarize findings. Output only the facts needed for implementation."
);

// 编码者：只负责根据需求和研究笔记写代码
const coder = createSpecialist(
  "coder",
  "You are a senior TypeScript developer. Given requirements and research notes, write clean, tested code. Nothing else."
);

// 审阅者：只负责找缺陷
const reviewer = createSpecialist(
  "reviewer",
  "You are a code reviewer. Find bugs, security issues, and logic errors. Be specific. Cite line numbers."
);
```

每个专家都有专注的提示词。每个都获得干净的上下文窗口，只包含它需要的输入。

### 步骤 3：通过消息进行协调

用显式消息传递将专家们连接起来：

```typescript
type AgentMessage = {
  from: string;
  to: string;
  content: string;
  timestamp: number;
};

async function multiAgentApproach(task: string): Promise<AgentResult> {
  const messages: AgentMessage[] = [];
  let totalTokens = 0;
  let totalToolCalls = 0;

  // 研究员执行研究
  const researchResult = await researcher.run(task);
  messages.push({
    from: "researcher",
    to: "coder",
    content: researchResult.content,
    timestamp: Date.now(),
  });
  totalTokens += researchResult.tokensUsed;
  totalToolCalls += researchResult.toolCalls;

  // 编码者只接收发给自己的消息
  const coderInput = messages
    .filter((m) => m.to === "coder")
    .map((m) => `[From ${m.from}]: ${m.content}`)
    .join("\n");

  const codeResult = await coder.run(coderInput);
  messages.push({
    from: "coder",
    to: "reviewer",
    content: codeResult.content,
    timestamp: Date.now(),
  });
  totalTokens += codeResult.tokensUsed;
  totalToolCalls += codeResult.toolCalls;

  // 审阅者只接收发给自己的消息
  const reviewerInput = messages
    .filter((m) => m.to === "reviewer")
    .map((m) => `[From ${m.from}]: ${m.content}`)
    .join("\n");

  const reviewResult = await reviewer.run(reviewerInput);
  messages.push({
    from: "reviewer",
    to: "orchestrator",
    content: reviewResult.content,
    timestamp: Date.now(),
  });
  totalTokens += reviewResult.tokensUsed;
  totalToolCalls += reviewResult.toolCalls;

  return {
    content: messages.map((m) => `[${m.from} -> ${m.to}]: ${m.content}`).join("\n\n"),
    tokensUsed: totalTokens,
    toolCalls: totalToolCalls,
  };
}
```

每个智能体只接收发给它的消息。没有上下文污染。研究员 5 万 token 的文档阅读永远不会进入审阅者的上下文。

### 步骤 4：对比

```typescript
async function compare() {
  const task = "Build a rate limiter middleware for an Express.js API";

  console.log("=== 单智能体 ===");
  const single = await singleAgentApproach(task);
  console.log(`Token 消耗: ${single.tokensUsed}`);
  console.log(`工具调用次数: ${single.toolCalls}`);

  console.log("\n=== 多智能体 ===");
  const multi = await multiAgentApproach(task);
  console.log(`Token 消耗: ${multi.tokensUsed}`);
  console.log(`工具调用次数: ${multi.toolCalls}`);
}
```

多智能体版本使用更多总 token（三个智能体，三次独立的 LLM 调用），但每个智能体的上下文保持干净。由于系统提示词是专门化的，每个阶段的质量都会提高。

## 实际应用

本课产出一个可复用的提示词，用于决定何时采用多智能体。参见 `outputs/prompt-multi-agent-decision.md`。

## 练习

1. 添加第四个专家："测试者"智能体，接收编码者的代码和审阅者的反馈，然后编写测试
2. 修改流水线，使审阅者可以将反馈发回编码者进行修订循环（最多 2 轮）
3. 将串行流水线转换为扇出模式：并行运行研究员和"需求分析"智能体，然后在传给编码者之前合并它们的输出

## 关键术语

| 术语 | 常见说法 | 实际含义 |
|------|---------|---------|
| Swarm（群体） | "AI 智能体蜂群思维" | 一组具有共享状态且没有固定领导者的对等智能体。行为从局部交互中涌现。 |
| Orchestrator（编排器） | "老板智能体" | 其工具包括生成和管理其他智能体的智能体。它负责规划和委派，但可能不执行实际工作。 |
| Coordinator（协调器） | "交通警察" | 一个非智能体组件（通常只是代码，不是 LLM），根据规则在智能体之间路由消息。 |
| Consensus（共识） | "智能体们达成一致" | 一种协议，要求多个智能体在继续之前必须达成一致。当冲突输出需要解决时使用。 |
| Emergent behavior（涌现行为） | "智能体们自己想出来的" | 从智能体交互中产生的系统级模式，但并未被显式编程。可能是有益的或有害的。 |
| Fan-out / fan-in（扇出/扇入） | "智能体的 MapReduce" | 将任务拆分给并行智能体（扇出），然后合并它们的结果（扇入）。 |
| Message passing（消息传递） | "智能体互相交谈" | 智能体之间的通信机制：结构化数据从一个智能体发送到另一个，替代共享上下文窗口。 |

## 延伸阅读

- [The Landscape of Emerging AI Agent Architectures](https://arxiv.org/abs/2409.02977) - 多智能体模式综述
- [AutoGen: Enabling Next-Gen LLM Applications](https://arxiv.org/abs/2308.08155) - 微软的多智能体对话框架
- [Claude Code subagents documentation](https://docs.anthropic.com/en/docs/claude-code) - Claude Code 如何通过 Task 进行委派
- [CrewAI documentation](https://docs.crewai.com/) - 基于角色的多智能体框架