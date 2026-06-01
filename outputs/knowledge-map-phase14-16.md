# Phase 14–16 知识图谱：Agent 工程 → 自主系统 → 多智能体与群体

> 本图谱覆盖 **89 课时**，从单智能体核心能力，到自主进化与安全边界，再到多智能体协作与群体智能。
> 所有 Mermaid 代码可直接粘贴到支持 Mermaid 的渲染器中查看（GitHub / VS Code / Notion / Obsidian）。

---

## 目录

1. [全景总览](#1-全景总览)
2. [Phase 14：Agent Engineering 详细图谱](#2-phase-14agent-engineering-详细图谱)
3. [Phase 15：Autonomous Systems 详细图谱](#3-phase-15autonomous-systems-详细图谱)
4. [Phase 16：Multi-Agent & Swarms 详细图谱](#4-phase-16multi-agent--swarms-详细图谱)
5. [跨阶段依赖关系](#5-跨阶段依赖关系)
6. [知识簇概览表](#6-知识簇概览表)
7. [建议学习路径](#7-建议学习路径)

---

## 1. 全景总览

```mermaid
graph TB
    subgraph P14["🧠 Phase 14: Agent Engineering<br/>42 课 · ~42h"]
        direction TB
        P14A["推理模式<br/>01-05"]
        P14B["工具与记忆<br/>06-11"]
        P14C["框架与 SDK<br/>12-18"]
        P14D["评测与可观测<br/>19-24"]
        P14E["多智能体初探<br/>25-28"]
        P14F["生产实践<br/>29-30"]
        P14G["Agent Workbench 专题<br/>31-42"]
        P14A --> P14B --> P14C
        P14C --> P14D
        P14D --> P14E --> P14F
        P14F --> P14G
    end

    subgraph P15["🚀 Phase 15: Autonomous Systems<br/>22 课 · ~20h"]
        direction TB
        P15A["自主推理与进化<br/>01-08"]
        P15B["自主编码 Agent<br/>09-11"]
        P15C["安全控制层<br/>12-17"]
        P15D["安全框架与评估<br/>18-22"]
        P15A --> P15B --> P15C --> P15D
    end

    subgraph P16["🕸️ Phase 16: Multi-Agent & Swarms<br/>25 课 · ~28h"]
        direction TB
        P16A["基础与通信<br/>01-04"]
        P16B["架构模式<br/>05-11"]
        P16C["协调机制<br/>12-16"]
        P16D["高级模拟与学习<br/>17-21"]
        P16E["生产与评估<br/>22-25"]
        P16A --> P16B --> P16C
        P16C --> P16D --> P16E
    end

    P14 -->|"推理 + 工具 + 记忆基础"| P15
    P14 -->|"框架 + 多智能体初探"| P16
    P15 -->|"安全边界 + 控制机制"| P16

    style P14 fill:#1a1a2e,stroke:#e94560,color:#fff
    style P15 fill:#16213e,stroke:#0f3460,color:#fff
    style P16 fill:#1b262c,stroke:#3282b8,color:#fff
```

---

## 2. Phase 14：Agent Engineering 详细图谱

### 2.1 推理模式（课程 01-05）

```mermaid
graph LR
    AL["01 Agent Loop<br/>感知→推理→行动 循环"] --> RW["02 ReWOO<br/>Plan-and-Execute<br/>先规划再执行"]
    RW --> RF["03 Reflexion<br/>语言强化学习<br/>自我反思改进"]
    RF --> TOT["04 Tree of Thoughts<br/>& LATS<br/>树搜索推理"]
    TOT --> SR["05 Self-Refine<br/>& CRITIC<br/>迭代自我批评"]

    style AL fill:#2d3436,color:#dfe6e9
    style RW fill:#636e72,color:#dfe6e9
    style RF fill:#b2bec3,color:#2d3436
    style TOT fill:#dfe6e9,color:#2d3436
    style SR fill:#ffeaa7,color:#2d3436
```

**核心思想**：从最简单的循环（Observe-Think-Act），逐步引入规划、反思、树搜索、自批评等高级推理策略。

### 2.2 工具与记忆（课程 06-11）

```mermaid
graph TD
    TU["06 Tool Use<br/>& Function Calling"] --> MC["07 Memory: Virtual Context<br/>& MemGPT<br/>虚拟上下文管理"]
    MC --> LB["08 Memory Blocks<br/>& Sleep-Time Compute<br/>(Letta)<br/>异步记忆整理"]
    LB --> HM["09 Hybrid Memory<br/>Vector + Graph + KV<br/>(Mem0)"]
    TU --> SL["10 Skill Libraries<br/>& Lifelong Learning<br/>(Voyager)"]
    HM --> PL["11 Planning: HTN<br/>& Evolutionary Search<br/>分层任务网络"]

    style TU fill:#00b894,color:#fff
    style MC fill:#00cec9,color:#fff
    style LB fill:#0984e3,color:#fff
    style HM fill:#6c5ce7,color:#fff
    style SL fill:#fdcb6e,color:#2d3436
    style PL fill:#e17055,color:#fff
```

**核心思想**：工具是 Agent 的手，记忆是 Agent 的大脑——从单工具调用到复合记忆系统，从即时技能到终身学习。

### 2.3 框架与 SDK（课程 12-18）

```mermaid
graph LR
    AP["12 Anthropic<br/>Workflow Patterns<br/>编排模式先驱"] --> LG["13 LangGraph<br/>有状态图<br/>& 持久执行"]
    LG --> AG["14 AutoGen v0.4<br/>Actor 模型<br/>异步消息"]
    LG --> CR["15 CrewAI<br/>角色化团队<br/>& Flows"]
    AG --> OA["16 OpenAI<br/>Agents SDK<br/>Handoffs<br/>Guardrails"]
    CR --> CA["17 Claude<br/>Agent SDK<br/>Subagents<br/>Session Store"]
    OA --> AM["18 Agno & Mastra<br/>生产级运行时"]
    CA --> AM

    style AP fill:#fd79a8,color:#fff
    style LG fill:#e84393,color:#fff
    style AG fill:#a29bfe,color:#2d3436
    style CR fill:#74b9ff,color:#2d3436
    style OA fill:#55efc4,color:#2d3436
    style CA fill:#ffeaa7,color:#2d3436
    style AM fill:#dfe6e9,color:#2d3436
```

**核心思想**：理解主流 Agent 框架的设计哲学——图编排（LangGraph）、Actor 模型（AutoGen）、角色协作（CrewAI）、Handoff 模式（OpenAI SDK）。

### 2.4 评测与可观测（课程 19-24）

```mermaid
graph LR
    SB["19 SWE-bench<br/>GAIA<br/>AgentBench"] --> WA["20 WebArena<br/>OSWorld<br/>浏览器 / 桌面"]
    WA --> CU["21 Computer Use<br/>Claude · OpenAI CUA<br/>Gemini"]
    CU --> VA["22 Voice Agents<br/>Pipecat<br/>LiveKit"]
    VA --> OT["23 OpenTelemetry<br/>GenAI Conventions<br/>标准化追踪"]
    OT --> OB["24 Agent Observability<br/>Langfuse · Phoenix<br/>Opik"]

    style SB fill:#fdcb6e,color:#2d3436
    style WA fill:#ffeaa7,color:#2d3436
    style CU fill:#00b894,color:#fff
    style VA fill:#00cec9,color:#fff
    style OT fill:#0984e3,color:#fff
    style OB fill:#6c5ce7,color:#fff
```

**核心思想**：如何衡量 Agent 的能力？从代码基准（SWE-bench）到交互环境（WebArena），再到标准化可观测性。

### 2.5 多智能体初探 + 生产实践（课程 25-30）

```mermaid
graph TD
    MD["25 Multi-Agent<br/>Debate<br/>多 Agent 辩论"] --> FM["26 Failure Modes<br/>智能体为何失败"]
    FM --> PI["27 Prompt Injection<br/>& PVE Defense<br/>安全防御"]
    PI --> OP["28 Orchestration<br/>Supervisor · Swarm<br/>Hierarchical"]
    OP --> PR["29 Production Runtimes<br/>Queue · Event · Cron"]
    PR --> ED["30 Eval-Driven<br/>Agent Development<br/>评测驱动开发"]

    style MD fill:#e17055,color:#fff
    style FM fill:#d63031,color:#fff
    style PI fill:#e84393,color:#fff
    style OP fill:#a29bfe,color:#2d3436
    style PR fill:#74b9ff,color:#2d3436
    style ED fill:#55efc4,color:#2d3436
```

### 2.6 Agent Workbench 专题（课程 31-42）

```mermaid
graph TD
    WM["31 Why Models Fail<br/>为什么有能力的模型<br/>仍然失败"] --> MW["32 Minimal<br/>Agent Workbench<br/>最小工作台"]
    MW --> IC["33 Instructions as<br/>Executable Constraints<br/>指令即约束"]
    IC --> RM["34 Repo Memory<br/>& Durable State<br/>仓库记忆"]
    RM --> IS["35 Initialization<br/>Scripts<br/>初始化脚本"]
    IS --> SC["36 Scope Contracts<br/>& Task Boundaries<br/>范围契约"]
    SC --> RF["37 Runtime Feedback<br/>Loops<br/>运行时反馈"]
    RF --> VG["38 Verification Gates<br/>验证门"]
    VG --> RA["39 Reviewer Agent<br/>Builder 与 Marker<br/>分离"]
    RA --> MH["40 Multi-Session<br/>Handoff<br/>多会话交接"]
    MH --> RR["41 Workbench on<br/>Real Repos<br/>真实仓库实战"]
    RR --> CP["42 Capstone: Ship a<br/>Reusable Workbench Pack<br/>毕业设计"]

    style WM fill:#d63031,color:#fff
    style MW fill:#e17055,color:#fff
    style IC fill:#fdcb6e,color:#2d3436
    style RM fill:#00b894,color:#fff
    style IS fill:#00cec9,color:#fff
    style SC fill:#0984e3,color:#fff
    style RF fill:#6c5ce7,color:#fff
    style VG fill:#a29bfe,color:#2d3436
    style RA fill:#fd79a8,color:#fff
    style MH fill:#e84393,color:#fff
    style RR fill:#74b9ff,color:#2d3436
    style CP fill:#ffeaa7,color:#2d3436
```

**核心思想**：这是 Phase 14 的巅峰——12 节课逐步构建一个完整的 Agent 工作台，覆盖从"为什么失败"到"如何在真实仓库中可靠运行"的全链路。

---

## 3. Phase 15：Autonomous Systems 详细图谱

### 3.1 自主推理与进化（课程 01-08）

```mermaid
graph TD
    LH["01 Long-Horizon Agents<br/>从聊天机器人到<br/>长期自主 Agent<br/>(METR)"] --> ST["02 STaR / V-STaR<br/>/ Quiet-STaR<br/>自我教学推理"]
    ST --> AE["03 AlphaEvolve<br/>进化编码 Agent"]
    AE --> DG["04 Darwin Gödel Machine<br/>自修改 Agent"]
    DG --> AS["05 AI Scientist v2<br/>研讨会级自动研究"]
    AS --> AR["06 Automated Alignment<br/>Research (Anthropic AAR)<br/>自动对齐研究"]
    AR --> RS["07 Recursive<br/>Self-Improvement<br/>能力 vs 对齐"]
    RS --> BS["08 Bounded<br/>Self-Improvement<br/>有界自改进设计"]

    style LH fill:#2d3436,color:#dfe6e9
    style ST fill:#636e72,color:#dfe6e9
    style AE fill:#b2bec3,color:#2d3436
    style DG fill:#dfe6e9,color:#2d3436
    style AS fill:#ffeaa7,color:#2d3436
    style AR fill:#fdcb6e,color:#2d3436
    style RS fill:#e17055,color:#fff
    style BS fill:#d63031,color:#fff
```

**核心思想**：从"长时间运行"到"自我改进"——Agent 不再只是执行任务，而是进化自身。递归自改进是能力的顶峰，也是风险的起点。

### 3.2 自主编码 Agent（课程 09-11）

```mermaid
graph LR
    CA["09 Coding Agent<br/>Landscape<br/>SWE-bench · CodeAct<br/>编码 Agent 全景"] --> CC["10 Claude Code<br/>Permission Modes<br/>& Auto Mode<br/>权限模式"]
    CC --> BA["11 Browser Agents<br/>& Indirect<br/>Prompt Injection<br/>浏览器 Agent + 安全"]

    style CA fill:#00b894,color:#fff
    style CC fill:#0984e3,color:#fff
    style BA fill:#e84393,color:#fff
```

**核心思想**：自主 Agent 的两个核心应用场景——代码编写和浏览器操作，各自带来独特的安全挑战。

### 3.3 安全控制层（课程 12-17）

```mermaid
graph TD
    DE["12 Durable Execution<br/>长时间运行的<br/>持久执行"] --> CG["13 Cost Governors<br/>Action Budgets<br/>Iteration Caps<br/>成本控制器"]
    CG --> KS["14 Kill Switches<br/>Circuit Breakers<br/>Canary Tokens<br/>紧急停止机制"]
    KS --> PC["15 Propose-Then-Commit<br/>HITL<br/>人机协作审批"]
    PC --> CR["16 Checkpoints<br/>& Rollback<br/>检查点与回滚"]
    CR --> CO["17 Constitutional AI<br/>& Rule Overrides<br/>宪法 AI 与规则覆盖"]

    style DE fill:#0984e3,color:#fff
    style CG fill:#00b894,color:#fff
    style KS fill:#d63031,color:#fff
    style PC fill:#fdcb6e,color:#2d3436
    style CR fill:#6c5ce7,color:#fff
    style CO fill:#a29bfe,color:#2d3436
```

**核心思想**：自主性越强，安全控制越重要。这一层是"放手"与"拉缰"之间的工程平衡——成本封顶、紧急刹车、人工审批、状态回滚、规则约束。

### 3.4 安全框架与评估（课程 18-22）

```mermaid
graph LR
    LG["18 Llama Guard<br/>输入/输出分类<br/>安全守卫"] --> RSP["19 Anthropic<br/>RSP v3.0<br/>负责任扩展策略"]
    RSP --> PF["20 OpenAI Preparedness<br/>& DeepMind FSF<br/>前沿安全框架"]
    PF --> MT["21 METR Time Horizons<br/>& External Evaluation<br/>外部评估"]
    MT --> CI["22 CAIS · CAISI<br/>Societal-Scale Risk<br/>社会级风险"]

    style LG fill:#e84393,color:#fff
    style RSP fill:#fd79a8,color:#fff
    style PF fill:#a29bfe,color:#2d3436
    style MT fill:#74b9ff,color:#2d3436
    style CI fill:#dfe6e9,color:#2d3436
```

**核心思想**：从模型级安全（Llama Guard）到机构级策略（RSP/FSF），再到社会级风险评估（CAIS/CAISI），理解安全的多个层次。

---

## 4. Phase 16：Multi-Agent & Swarms 详细图谱

### 4.1 基础与通信（课程 01-04）

```mermaid
graph LR
    WM["01 Why Multi-Agent<br/>为什么需要<br/>多智能体"] --> FP["02 FIPA-ACL Heritage<br/>& Speech Acts<br/>言语行为理论"]
    FP --> CP["03 Communication<br/>Protocols<br/>通信协议"]
    CP --> PM["04 Multi-Agent<br/>Primitive Model<br/>原语模型"]

    style WM fill:#2d3436,color:#dfe6e9
    style FP fill:#636e72,color:#dfe6e9
    style CP fill:#b2bec3,color:#2d3436
    style PM fill:#dfe6e9,color:#2d3436
```

**核心思想**：多智能体不是简单地复制单个 Agent——它需要通信协议、言语行为理论和统一的原语模型作为基础。

### 4.2 架构模式（课程 05-11）

```mermaid
graph TD
    SV["05 Supervisor /<br/>Orchestrator-Worker<br/>主管-工人模式"] --> HI["06 Hierarchical<br/>Architecture<br/>层级架构<br/>& 分解漂移"]
    HI --> SM["07 Society of Mind<br/>& Multi-Agent Debate<br/>心智社会与辩论"]
    SM --> RS["08 Role Specialization<br/>Planner · Critic<br/>Executor · Verifier<br/>角色分工"]
    RS --> PS["09 Parallel Swarm<br/>& Networked<br/>Architectures<br/>并行群体网络"]
    PS --> GC["10 Group Chat<br/>& Speaker Selection<br/>群聊与发言者选择"]
    GC --> HR["11 Handoffs & Routines<br/>Stateless Orchestration<br/>无状态编排"]

    style SV fill:#e17055,color:#fff
    style HI fill:#fdcb6e,color:#2d3436
    style SM fill:#00b894,color:#fff
    style RS fill:#00cec9,color:#fff
    style PS fill:#0984e3,color:#fff
    style GC fill:#6c5ce7,color:#fff
    style HR fill:#a29bfe,color:#2d3436
```

**核心思想**：7 种架构模式——从最简单的主管派活（Supervisor），到复杂的无状态编排（Handoffs），每种模式适用于不同场景。

### 4.3 协调机制（课程 12-16）

```mermaid
graph TD
    A2A["12 A2A Protocol<br/>Agent-to-Agent<br/>智能体间协议"] --> SH["13 Shared Memory<br/>& Blackboard<br/>共享记忆与黑板"]
    SH --> CB["14 Consensus & BFT<br/>拜占庭容错<br/>共识机制"]
    CB --> VT["15 Voting · Self-Consistency<br/>& Debate Topology<br/>投票与辩论拓扑"]
    VT --> NG["16 Negotiation<br/>& Bargaining<br/>谈判与博弈"]

    style A2A fill:#0984e3,color:#fff
    style SH fill:#00b894,color:#fff
    style CB fill:#e17055,color:#fff
    style VT fill:#fdcb6e,color:#2d3436
    style NG fill:#e84393,color:#fff
```

**核心思想**：当多个 Agent 需要达成一致时，需要协议（A2A）、共享状态（Blackboard）、容错共识（BFT）、投票决策和谈判博弈。

### 4.4 高级模拟与学习（课程 17-21）

```mermaid
graph TD
    GA["17 Generative Agents<br/>& Emergent Simulation<br/>生成式 Agent 模拟"] --> TM["18 Theory of Mind<br/>& Emergent Coordination<br/>心智理论协调"]
    TM --> SW["19 Swarm Optimization<br/>PSO · ACO<br/>群体智能优化"]
    SW --> MR["20 MARL<br/>MADDPG · QMIX<br/>MAPPO<br/>多智能体强化学习"]
    MR --> AE["21 Agent Economies<br/>Token Incentives<br/>Reputation<br/>智能体经济学"]

    style GA fill:#6c5ce7,color:#fff
    style TM fill:#a29bfe,color:#2d3436
    style SW fill:#fdcb6e,color:#2d3436
    style MR fill:#e17055,color:#fff
    style AE fill:#d63031,color:#fff
```

**核心思想**：从模拟人类社会（Generative Agents），到经典群体优化（PSO/ACO），再到深度多智能体强化学习（MARL），最终引入经济激励机制。

### 4.5 生产与评估（课程 22-25）

```mermaid
graph LR
    PS["22 Production Scaling<br/>Queues · Checkpoints<br/>Durability<br/>生产扩展"] --> FM["23 Failure Modes<br/>MAST · Groupthink<br/>Monoculture<br/>Cascading<br/>失败模式"]
    FM --> EV["24 Evaluation &<br/>Coordination<br/>Benchmarks<br/>协调基准测试"]
    EV --> CS["25 Case Studies &<br/>2026 State of the Art<br/>案例与最新进展"]

    style PS fill:#00b894,color:#fff
    style FM fill:#d63031,color:#fff
    style EV fill:#0984e3,color:#fff
    style CS fill:#ffeaa7,color:#2d3436
```

---

## 5. 跨阶段依赖关系

```mermaid
graph TB
    subgraph P14["Phase 14: Agent Engineering"]
        T14_06["06 Tool Use"]
        T14_07["07-09 Memory Systems"]
        T14_25["25 Multi-Agent Debate"]
        T14_26["26 Failure Modes"]
        T14_28["28 Orchestration Patterns"]
        T14_G["31-42 Agent Workbench"]
    end

    subgraph P15["Phase 15: Autonomous Systems"]
        T15_11["11 Browser Agents"]
        T15_12["12-17 Safety Controls"]
        T15_17["17 Constitutional AI"]
        T15_18["18-22 Safety Frameworks"]
    end

    subgraph P16["Phase 16: Multi-Agent"]
        T16_05["05-06 Supervisor/Hierarchical"]
        T16_07["07 Society of Mind"]
        T16_12["12 A2A Protocol"]
        T16_13["13 Shared Memory"]
        T16_14["14 Consensus/BFT"]
        T16_15["15 Voting/Debate Topology"]
        T16_22["22 Production Scaling"]
        T16_23["23 Failure Modes"]
    end

    T14_06 -->|"工具调用 → 浏览器操作"| T15_11
    T14_07 -->|"单 Agent 记忆 → 共享记忆"| T16_13
    T14_25 -->|"辩论初探 → 完整辩论拓扑"| T16_15
    T14_25 -->|"辩论 → 心智社会"| T16_07
    T14_26 -->|"单 Agent 失败 → 群体失败"| T16_23
    T14_28 -->|"编排模式 → 层级架构"| T16_05
    T14_06 -->|"工具使用 → A2A 协议"| T16_12
    T15_17 -->|"宪法 AI → 共识机制"| T16_14
    T15_12 -->|"安全控制 → 生产扩展"| T16_22
    T14_G -->|"工作台经验 → 自主安全"| T15_12
    T15_18 -->|"安全框架 → 群体评估"| T16_23

    style P14 fill:#1a1a2e,stroke:#e94560,color:#fff
    style P15 fill:#16213e,stroke:#0f3460,color:#fff
    style P16 fill:#1b262c,stroke:#3282b8,color:#fff
```

---

## 6. 知识簇概览表

### Phase 14：Agent Engineering（智能体工程）

| 知识簇 | 课程编号 | 核心主题 | 关键概念 |
|--------|---------|---------|---------|
| 推理模式 | 01-05 | Agent 推理策略 | Agent Loop, ReWOO, Reflexion, ToT/LATS, Self-Refine, CRITIC |
| 工具与记忆 | 06-11 | 外部能力接入 | Function Calling, MemGPT, Letta, Mem0, Voyager, HTN, 进化搜索 |
| 框架与 SDK | 12-18 | 工程化框架 | Anthropic Patterns, LangGraph, AutoGen, CrewAI, OpenAI SDK, Claude SDK, Agno, Mastra |
| 评测与可观测 | 19-24 | 质量保障 | SWE-bench, GAIA, WebArena, OSWorld, Computer Use, Voice Agents, OTel, Langfuse |
| 多智能体初探 | 25-28 | 协作与安全 | Multi-Agent Debate, Failure Modes, Prompt Injection, Orchestration |
| 生产实践 | 29-30 | 上线运维 | Production Runtimes, Eval-Driven Dev |
| Agent Workbench | 31-42 | 工作台全流程 | 可执行约束, 仓库记忆, 范围契约, 验证门, Reviewer Agent, Capstone |

### Phase 15：Autonomous Systems（自主系统）

| 知识簇 | 课程编号 | 核心主题 | 关键概念 |
|--------|---------|---------|---------|
| 自主推理与进化 | 01-08 | 能力递进 | Long-Horizon, STaR, AlphaEvolve, Darwin Gödel, AI Scientist, AAR, 递归/有界自改进 |
| 自主编码 Agent | 09-11 | 代码与浏览器 | SWE-bench, CodeAct, Claude Code, Browser Agents, 间接注入 |
| 安全控制层 | 12-17 | 控制机制 | Durable Execution, Cost Governors, Kill Switches, HITL, Checkpoints, Constitutional AI |
| 安全框架与评估 | 18-22 | 制度保障 | Llama Guard, RSP, Preparedness/FSF, METR, CAIS/CAISI |

### Phase 16：Multi-Agent & Swarms（多智能体与群体）

| 知识簇 | 课程编号 | 核心主题 | 关键概念 |
|--------|---------|---------|---------|
| 基础与通信 | 01-04 | 理论根基 | Why Multi-Agent, FIPA-ACL, Speech Acts, 通信协议, 原语模型 |
| 架构模式 | 05-11 | 拓扑设计 | Supervisor, Hierarchical, Society of Mind, Role Specialization, Parallel Swarm, Group Chat, Handoffs |
| 协调机制 | 12-16 | 一致性达成 | A2A, Shared Memory, Consensus/BFT, Voting, Negotiation |
| 高级模拟与学习 | 17-21 | 涌现与优化 | Generative Agents, Theory of Mind, PSO/ACO, MARL, Agent Economies |
| 生产与评估 | 22-25 | 落地实战 | Production Scaling, Failure Modes, Benchmarks, Case Studies 2026 |

---

## 7. 建议学习路径

### 路径 A：快速入门（推荐，~30h）

如果你想快速掌握 Agent 工程的核心能力，按此路径学习：

```
Phase 14 推理模式 (01-05)
    ↓
Phase 14 工具与记忆 (06-11)
    ↓
Phase 14 框架选一 (13 LangGraph 或 16 OpenAI SDK)
    ↓
Phase 15 安全控制层 (12-17) ← 理解安全边界
    ↓
Phase 16 架构模式 (05-11) ← 理解多 Agent 编排
    ↓
Phase 14 Agent Workbench (31-42) ← 综合实战
```

### 路径 B：全面深入（完整，~90h）

按顺序完成所有 89 课时：

```
Phase 14 全部 (01-42)    ← 建立完整的 Agent 工程基础
    ↓
Phase 15 全部 (01-22)    ← 理解自主性与安全边界
    ↓
Phase 16 全部 (01-25)    ← 掌握多智能体协作与群体智能
```

### 路径 C：按主题横向学习

| 主题 | 路径 |
|------|------|
| **记忆系统** | P14-07 → P14-08 → P14-09 → P16-13 |
| **安全与对齐** | P14-27 → P15-14 → P15-15 → P15-17 → P15-18 → P16-14 |
| **编排模式** | P14-28 → P16-05 → P16-06 → P16-07 → P16-08 → P16-11 |
| **评测基准** | P14-19 → P14-20 → P15-21 → P16-24 |
| **生产落地** | P14-29 → P15-12 → P16-22 |
| **自改进** | P15-02 → P15-03 → P15-04 → P15-05 → P15-06 → P15-07 → P15-08 |

---

## 附录：关键术语速查

| 术语 | 含义 | 首次出现 |
|------|------|---------|
| Agent Loop | 感知→推理→行动的循环 | P14-01 |
| ReWOO | Reasoning Without Observation，先规划再执行 | P14-02 |
| Reflexion | 通过语言反思进行自我改进 | P14-03 |
| ToT | Tree of Thoughts，树搜索推理 | P14-04 |
| LATS | Language Agent Tree Search，语言 Agent 树搜索 | P14-04 |
| MemGPT | 虚拟上下文管理的 Agent 记忆系统 | P14-07 |
| HTN | Hierarchical Task Network，分层任务网络 | P14-11 |
| OTel | OpenTelemetry，可观测性标准 | P14-23 |
| STaR | Self-Taught Reasoner，自我教学推理 | P15-02 |
| AlphaEvolve | 进化式编码 Agent | P15-03 |
| Darwin Gödel Machine | 能自修改的 Agent 架构 | P15-04 |
| RSP | Responsible Scaling Policy，负责任扩展策略 | P15-19 |
| METR | Model Evaluation & Threat Research，模型评估与威胁研究 | P15-21 |
| FIPA-ACL | 基金会智能物理 Agent 通信语言 | P16-02 |
| A2A | Agent-to-Agent Protocol，智能体间协议 | P16-12 |
| BFT | Byzantine Fault Tolerance，拜占庭容错 | P16-14 |
| MARL | Multi-Agent Reinforcement Learning，多智能体强化学习 | P16-20 |
| PSO | Particle Swarm Optimization，粒子群优化 | P16-19 |
| ACO | Ant Colony Optimization，蚁群优化 | P16-19 |
| MAST | Multi-Agent Systemic Thinking failure，多智能体系统性失败 | P16-23 |