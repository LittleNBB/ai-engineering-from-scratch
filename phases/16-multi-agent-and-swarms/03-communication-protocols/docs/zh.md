# 通信协议（Communication Protocols）

> 说不同语言的智能体不是团队——他们是朝着虚空呐喊的陌生人。

**类型：** Build（构建）
**语言：** TypeScript
**前置课程：** Phase 14（Agent Engineering），Lesson 16.01（Why Multi-Agent）
**时间：** ~120 分钟

## 学习目标

- 实现 MCP 工具发现和调用，使智能体能够使用外部服务器暴露的工具
- 构建 A2A 智能体卡片（Agent Card）和任务端点，使一个智能体可以通过 HTTP 将工作委派给另一个智能体
- 比较 MCP（工具访问）、A2A（智能体间通信）、ACP（企业审计）和 ANP（去中心化信任），并解释哪个协议解决哪个问题
- 在一个系统中将多个协议连接在一起，使智能体通过 MCP 发现工具并通过 A2A 委派任务

## 问题所在

你把系统拆分成了多个智能体。一个研究员、一个编码者、一个审阅者。他们各自的工作都很出色。但现在你需要他们真正互相交流。

你的第一次尝试很明显：传递字符串。研究员返回一段文本，编码者尽可能地解析它。直到编码者误解了研究摘要，或者两个智能体互相等待导致死锁，或者你需要不同团队构建的智能体协作时，它才会出问题。突然间"只传字符串"就崩塌了。

这就是**通信协议问题**。如果没有关于智能体如何交换信息的共享契约，多智能体系统就是脆弱的、不可审计的，而且不可能扩展到超过你亲自编写的少数几个智能体。

AI 生态系统用四个协议来回应，每个解决问题的不同层面：

- **MCP** 用于工具访问
- **A2A** 用于智能体间协作
- **ACP** 用于企业可审计性
- **ANP** 用于去中心化身份和信任

本课深入讲解。你将阅读每个规范的真实线路格式，构建可工作的实现，并将四个协议连接成一个统一的系统。

## 核心概念

### 协议版图

将这四个协议想象为层级，每个解决一个不同的问题：

```mermaid
block-beta
  columns 1
  block:ANP["ANP — 智能体如何信任陌生人？\n去中心化身份（DID）、E2EE、元协议"]
  end
  block:A2A["A2A — 智能体如何在目标上协作？\nAgent Cards、任务生命周期、流式传输、协商"]
  end
  block:ACP["ACP — 智能体如何在可审计系统中通信？\n运行（Runs）、轨迹元数据、会话连续性"]
  end
  block:MCP["MCP — 智能体如何使用工具？\n工具发现、执行、上下文共享"]
  end

  style ANP fill:#f3e8ff,stroke:#7c3aed
  style A2A fill:#dbeafe,stroke:#2563eb
  style ACP fill:#fef3c7,stroke:#d97706
  style MCP fill:#d1fae5,stroke:#059669
```

它们不是竞争对手。它们在不同层级解决不同问题。

### MCP（回顾）

MCP 在 Phase 13 中有深入讲解。快速回顾：MCP 标准化了 LLM 连接外部工具和数据源的方式。它是一个**客户端-服务器**协议，智能体（客户端）发现并调用服务器暴露的工具。

```mermaid
sequenceDiagram
    participant Agent as 智能体（客户端）
    participant MCP1 as MCP 服务器<br/>（数据库、API、文件）

    Agent->>MCP1: 列出工具
    MCP1-->>Agent: 工具定义
    Agent->>MCP1: 调用工具 X
    MCP1-->>Agent: 结果
```

MCP 是**智能体到工具**的通信。它不帮助智能体互相交谈。

### A2A（Agent2Agent 协议）

**创建者：** Google（现归 Linux Foundation，标识为 `lf.a2a.v1`）
**规范版本：** 1.0.0
**问题：** 自主智能体如何协作、协商和互相委派任务？

A2A 是用于**点对端智能体协作**的协议。MCP 连接智能体到工具，A2A 连接智能体到其他智能体。每个智能体在一个知名 URL 上发布一个**Agent Card（智能体卡片）**，其他智能体发现它、与它协商并委派任务。

#### A2A 如何工作

```mermaid
sequenceDiagram
    participant Client as 客户端智能体
    participant Remote as 远程智能体

    Client->>Remote: GET /.well-known/agent-card.json
    Remote-->>Client: Agent Card（技能、模式、安全性）

    Client->>Remote: POST /message:send
    Remote-->>Client: 任务（submitted/working）

    alt 轮询
        Client->>Remote: GET /tasks/{id}
        Remote-->>Client: 任务状态 + 工件
    else 流式传输
        Client->>Remote: POST /message:stream
        Remote-->>Client: SSE: statusUpdate
        Remote-->>Client: SSE: artifactUpdate
        Remote-->>Client: SSE: completed
    end
```

#### 真实的 Agent Card

这是 A2A Agent Card 在实际中的样子。通过 `GET /.well-known/agent-card.json` 提供：

```json
{
  "name": "Research Agent",
  "description": "Searches documentation and summarizes findings",
  "version": "1.0.0",
  "supportedInterfaces": [
    {
      "url": "https://research-agent.example.com/a2a/v1",
      "protocolBinding": "JSONRPC",
      "protocolVersion": "1.0"
    },
    {
      "url": "https://research-agent.example.com/a2a/rest",
      "protocolBinding": "HTTP+JSON",
      "protocolVersion": "1.0"
    }
  ],
  "provider": {
    "organization": "Your Company",
    "url": "https://example.com"
  },
  "capabilities": {
    "streaming": true,
    "pushNotifications": false
  },
  "defaultInputModes": ["text/plain", "application/json"],
  "defaultOutputModes": ["text/plain", "application/json"],
  "skills": [
    {
      "id": "web-research",
      "name": "Web Research",
      "description": "Searches the web and synthesizes findings",
      "tags": ["research", "search", "summarization"],
      "examples": ["Research the latest changes in React 19"]
    },
    {
      "id": "doc-analysis",
      "name": "Documentation Analysis",
      "description": "Reads and analyzes technical documentation",
      "tags": ["docs", "analysis"],
      "inputModes": ["text/plain", "application/pdf"],
      "outputModes": ["application/json"]
    }
  ],
  "securitySchemes": {
    "bearer": {
      "httpAuthSecurityScheme": {
        "scheme": "Bearer",
        "bearerFormat": "JWT"
      }
    }
  },
  "security": [{ "bearer": [] }]
}
```

关键要点：
- **Skills（技能）**是智能体能做的事情。每个都有 ID、标签和支持的输入/输出 MIME 类型。这是客户端智能体判断远程智能体是否能处理其请求的方式。
- **supportedInterfaces** 列出多个协议绑定。单个智能体可以同时支持 JSON-RPC、REST 和 gRPC。
- **安全性**内置于卡片中。客户端在发出第一个请求之前就知道需要什么认证。

#### 任务生命周期

任务是 A2A 中的核心工作单元。它们通过定义好的状态流转：

```mermaid
stateDiagram-v2
    [*] --> submitted
    submitted --> working
    working --> input_required: 需要更多信息
    input_required --> working: 客户端发送数据
    working --> completed: 成功
    working --> failed: 错误
    working --> canceled: 客户端取消
    submitted --> rejected: 智能体拒绝

    completed --> [*]
    failed --> [*]
    canceled --> [*]
    rejected --> [*]

    note right of completed: 终态是不可变的。\n后续操作在同一 contextId\n内创建新任务。
```

全部 8 个状态（规范还定义了 `UNSPECIFIED` 作为哨兵值，此处省略）：

| 状态 | 终态？ | 含义 |
|---|---|---|
| `TASK_STATE_SUBMITTED` | 否 | 已确认，尚未处理 |
| `TASK_STATE_WORKING` | 否 | 正在处理中 |
| `TASK_STATE_INPUT_REQUIRED` | 否 | 智能体需要客户端更多信息 |
| `TASK_STATE_AUTH_REQUIRED` | 否 | 需要认证 |
| `TASK_STATE_COMPLETED` | 是 | 成功完成 |
| `TASK_STATE_FAILED` | 是 | 以错误结束 |
| `TASK_STATE_CANCELED` | 是 | 完成前被取消 |
| `TASK_STATE_REJECTED` | 是 | 智能体拒绝了任务 |

一旦任务到达终态，它就是不可变的。没有后续消息。后续操作在同一 `contextId` 内创建新任务。

#### 线路格式

A2A 使用 JSON-RPC 2.0。以下是真实的消息交换：

**客户端发送任务：**
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "SendMessage",
  "params": {
    "message": {
      "messageId": "msg-001",
      "role": "ROLE_USER",
      "parts": [{ "text": "Research React 19 compiler features" }]
    },
    "configuration": {
      "acceptedOutputModes": ["text/plain", "application/json"],
      "historyLength": 10
    }
  }
}
```

**智能体回复任务：**
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "task": {
      "id": "task-abc-123",
      "contextId": "ctx-xyz-789",
      "status": {
        "state": "TASK_STATE_COMPLETED",
        "timestamp": "2026-03-27T10:30:00Z"
      },
      "artifacts": [
        {
          "artifactId": "art-001",
          "name": "research-results",
          "parts": [{
            "data": {
              "findings": [
                "React 19 compiler auto-memoizes components",
                "No more manual useMemo/useCallback needed",
                "Compiler runs at build time, not runtime"
              ]
            },
            "mediaType": "application/json"
          }]
        }
      ]
    }
  }
}
```

**通过 SSE 流式传输：**
```text
POST /message:stream HTTP/1.1
Content-Type: application/json
A2A-Version: 1.0

data: {"task":{"id":"task-123","status":{"state":"TASK_STATE_WORKING"}}}

data: {"statusUpdate":{"taskId":"task-123","status":{"state":"TASK_STATE_WORKING","message":{"role":"ROLE_AGENT","parts":[{"text":"Searching documentation..."}]}}}}

data: {"artifactUpdate":{"taskId":"task-123","artifact":{"artifactId":"art-1","parts":[{"text":"partial findings..."}]},"append":true,"lastChunk":false}}

data: {"statusUpdate":{"taskId":"task-123","status":{"state":"TASK_STATE_COMPLETED"}}}
```

### ACP（智能体通信协议）

**创建者：** IBM / BeeAI
**规范版本：** 0.2.0（OpenAPI 3.1.1）
**状态：** 正在合并到 A2A（Linux Foundation 下）
**问题：** 智能体如何在具有完全可审计性、会话连续性和轨迹跟踪的情况下通信？

ACP 是**企业协议**。与许多总结所声称的不同，ACP **不**使用 JSON-LD。它是一个通过 OpenAPI 定义的直接的 REST/JSON API。它的特别之处在于**TrajectoryMetadata（轨迹元数据）**：每个智能体响应都可以携带产生该响应的推理步骤和工具调用的详细日志。

```mermaid
sequenceDiagram
    participant Client as 客户端
    participant ACP as ACP 智能体
    participant Audit as 审计日志

    Client->>ACP: POST /runs（模式: sync）
    ACP->>ACP: 处理请求...
    ACP->>Audit: 记录轨迹：<br/>推理 + 工具调用
    ACP-->>Client: 响应 + TrajectoryMetadata
    Note over Audit: 每一步都被记录：<br/>tool_name, tool_input,<br/>tool_output, reasoning
```

#### ACP 中的智能体发现

ACP 定义了四种发现方法：

```mermaid
graph LR
    A[智能体发现] --> B["运行时<br/>GET /agents"]
    A --> C["开放<br/>.well-known/agent.yml"]
    A --> D["注册表<br/>集中式目录"]
    A --> E["嵌入式<br/>容器标签"]

    style B fill:#dbeafe,stroke:#2563eb
    style C fill:#d1fae5,stroke:#059669
    style D fill:#fef3c7,stroke:#d97706
    style E fill:#f3e8ff,stroke:#7c3aed
```

**AgentManifest（智能体清单）**比 A2A 的 Agent Card 更简单：

```json
{
  "name": "summarizer",
  "description": "Summarizes documents with source citations",
  "input_content_types": ["text/plain", "application/pdf"],
  "output_content_types": ["text/plain", "application/json"],
  "metadata": {
    "tags": ["summarization", "RAG"],
    "framework": "BeeAI",
    "capabilities": [
      {
        "name": "Document Summarization",
        "description": "Condenses long documents into key points"
      }
    ],
    "recommended_models": ["llama3.3:70b-instruct-fp16"],
    "license": "Apache-2.0",
    "programming_language": "Python"
  }
}
```

#### 运行生命周期

ACP 使用"Runs（运行）"而非"Tasks（任务）"。一次运行是一个智能体执行，有三种模式：

| 模式 | 行为 |
|---|---|
| `sync` | 阻塞式。响应包含完整结果。 |
| `async` | 立即返回 202。轮询 `GET /runs/{id}` 获取状态。 |
| `stream` | SSE 流。智能体工作时触发事件。 |

```mermaid
stateDiagram-v2
    [*] --> created
    created --> in_progress
    in_progress --> completed: 成功
    in_progress --> failed: 错误
    in_progress --> awaiting: 需要输入
    awaiting --> in_progress: 客户端恢复
    in_progress --> cancelling: 取消请求
    cancelling --> cancelled

    completed --> [*]
    failed --> [*]
    cancelled --> [*]
```

#### TrajectoryMetadata（审计追踪）

这是 ACP 的关键差异化特性。每个消息部分都可以包含元数据，精确展示智能体做了什么：

```json
{
  "role": "agent/researcher",
  "parts": [
    {
      "content_type": "text/plain",
      "content": "The weather in San Francisco is 72F and sunny.",
      "metadata": {
        "kind": "trajectory",
        "message": "I need to check the weather for this location",
        "tool_name": "weather_api",
        "tool_input": { "location": "San Francisco, CA" },
        "tool_output": { "temperature": 72, "condition": "sunny" }
      }
    }
  ]
}
```

对于受监管行业来说，这是黄金般的价值。每个答案都附带可证明的推理链：调用了哪些工具、使用了什么输入、收到了什么输出。没有黑箱。

ACP 还支持 **CitationMetadata（引用元数据）**用于来源归因：

```json
{
  "kind": "citation",
  "start_index": 0,
  "end_index": 47,
  "url": "https://weather.gov/sf",
  "title": "NWS San Francisco Forecast"
}
```

### ANP（智能体网络协议）

**创建者：** 开源社区（由高维昌发起）
**仓库：** [github.com/agent-network-protocol/AgentNetworkProtocol](https://github.com/agent-network-protocol/AgentNetworkProtocol)
**问题：** 来自不同组织的智能体如何在没有中央权威的情况下相互信任？

ANP 是**去中心化身份协议**。它使用 W3C 去中心化标识符（DID）和端到端加密来建立信任。与通过已知端点发现智能体的 A2A 不同，ANP 让智能体通过密码学证明其身份。

ANP 有三层：

```mermaid
graph TB
    subgraph Layer3["第 3 层：应用协议"]
        AD[智能体描述文档]
        DISC[发现端点]
    end
    subgraph Layer2["第 2 层：元协议"]
        NEG[AI 驱动的协议协商]
        CODE[动态代码生成]
    end
    subgraph Layer1["第 1 层：身份与安全通信"]
        DID["did:wba（W3C DID）"]
        HPKE[HPKE E2EE - RFC 9180]
        SIG[签名验证]
    end

    Layer3 --> Layer2
    Layer2 --> Layer1

    style Layer1 fill:#d1fae5,stroke:#059669
    style Layer2 fill:#dbeafe,stroke:#2563eb
    style Layer3 fill:#f3e8ff,stroke:#7c3aed
```

#### DID 文档（真实结构）

ANP 使用一种名为 `did:wba`（Web-Based Agent，基于 Web 的智能体）的自定义 DID 方法。DID `did:wba:example.com:user:alice` 解析为 `https://example.com/user/alice/did.json`：

```json
{
  "@context": [
    "https://www.w3.org/ns/did/v1",
    "https://w3id.org/security/suites/jws-2020/v1",
    "https://w3id.org/security/suites/secp256k1-2019/v1"
  ],
  "id": "did:wba:example.com:user:alice",
  "verificationMethod": [
    {
      "id": "did:wba:example.com:user:alice#key-1",
      "type": "EcdsaSecp256k1VerificationKey2019",
      "controller": "did:wba:example.com:user:alice",
      "publicKeyJwk": {
        "crv": "secp256k1",
        "x": "NtngWpJUr-rlNNbs0u-Aa8e16OwSJu6UiFf0Rdo1oJ4",
        "y": "qN1jKupJlFsPFc1UkWinqljv4YE0mq_Ickwnjgasvmo",
        "kty": "EC"
      }
    },
    {
      "id": "did:wba:example.com:user:alice#key-x25519-1",
      "type": "X25519KeyAgreementKey2019",
      "controller": "did:wba:example.com:user:alice",
      "publicKeyMultibase": "z9hFgmPVfmBZwRvFEyniQDBkz9LmV7gDEqytWyGZLmDXE"
    }
  ],
  "authentication": [
    "did:wba:example.com:user:alice#key-1"
  ],
  "keyAgreement": [
    "did:wba:example.com:user:alice#key-x25519-1"
  ],
  "humanAuthorization": [
    "did:wba:example.com:user:alice#key-1"
  ],
  "service": [
    {
      "id": "did:wba:example.com:user:alice#agent-description",
      "type": "AgentDescription",
      "serviceEndpoint": "https://example.com/agents/alice/ad.json"
    }
  ]
}
```

关键要点：
- **密钥分离**是强制的。签名密钥（secp256k1）与加密密钥（X25519）分开。
- **`humanAuthorization`** 是 ANP 独有的。这些密钥在使用前需要显式的人类批准（生物识别、密码、HSM）。高风险操作（如资金转移）通过此路径。
- **`keyAgreement`** 密钥用于 HPKE 端到端加密（RFC 9180）。
- **service** 部分链接到智能体描述文档。

#### ANP 中信任如何工作

ANP **不**使用信任网络或背书图。信任是双边的，每次交互都验证：

```mermaid
sequenceDiagram
    participant A as 智能体 A
    participant Domain as 智能体 A 的域名
    participant B as 智能体 B

    A->>B: HTTP 请求 + DID + 签名
    B->>Domain: 获取 DID 文档（HTTPS）
    Domain-->>B: DID 文档 + 公钥
    B->>B: 用公钥验证签名
    B-->>A: 签发访问令牌
    A->>B: 后续请求使用令牌
    Note over A,B: 信任 = TLS 域名验证<br/>+ DID 签名验证<br/>+ 最小信任原则
```

信任来自三个来源：
1. **域名级 TLS** 验证 DID 文档的宿主
2. **DID 密码学签名** 验证智能体的身份
3. **最小信任原则** 仅授予最小权限

没有基于八卦的信任传播或 PageRank 评分。你通过 DID 直接验证每个智能体。

#### 元协议协商（Meta-Protocol Negotiation）

这是 ANP 最具创新性的特性。当来自不同生态系统的两个智能体相遇时，它们不需要预先约定的数据格式。它们通过自然语言协商：

```json
{
  "action": "protocolNegotiation",
  "sequenceId": 0,
  "candidateProtocols": "I can communicate using:\n1. JSON-RPC with hotel booking schema\n2. REST with OpenAPI 3.1 spec\n3. Natural language over HTTP",
  "modificationSummary": "Initial proposal",
  "status": "negotiating"
}
```

```mermaid
sequenceDiagram
    participant A as 智能体 A
    participant B as 智能体 B

    A->>B: protocolNegotiation（候选协议）
    B->>A: protocolNegotiation（反提案）
    A->>B: protocolNegotiation（已接受）
    Note over A,B: 智能体动态生成代码<br/>来处理约定的格式。<br/>最多 10 轮，然后超时。
```

智能体来回交流（最多 10 轮）直到就格式达成一致，然后动态生成代码来处理它。状态值：`negotiating`（协商中）、`rejected`（拒绝）、`accepted`（接受）、`timeout`（超时）。

这意味着两个从未见过面的智能体可以在没有人预先定义共享 Schema 的情况下，自行解决如何通信的问题。

### 对比（修正版）

| | MCP | A2A | ACP | ANP |
|---|---|---|---|---|
| **创建者** | Anthropic | Google / Linux Foundation | IBM / BeeAI | 社区 |
| **规范格式** | JSON-RPC | JSON-RPC / REST / gRPC | OpenAPI 3.1（REST） | JSON-RPC |
| **主要用途** | 智能体到工具 | 智能体到智能体 | 智能体到智能体 | 智能体到智能体 |
| **发现** | 工具列表 | `/.well-known/agent-card.json` | `GET /agents`、`/.well-known/agent.yml` | `/.well-known/agent-descriptions`、DID 服务端点 |
| **身份** | 隐式（本地） | 安全方案（OAuth、mTLS） | 服务器级 | W3C DID（`did:wba`）+ E2EE |
| **审计追踪** | 无 | 基本（任务历史） | TrajectoryMetadata（工具调用、推理） | 未正式指定 |
| **状态机** | 无 | 9 个任务状态 | 7 个运行状态 | 无 |
| **流式传输** | 无 | SSE | SSE | 传输无关 |
| **独特特性** | 工具 Schema | Agent Cards + 技能 | 轨迹审计追踪 | 元协议协商 |
| **最适合** | 工具和数据 | 动态协作 | 受监管行业 | 跨组织信任 |
| **状态** | 稳定 | 稳定（v1.0） | 合并到 A2A | 积极开发中 |

### 它们如何协同工作

这些协议不是互斥的。一个真实的企业系统会使用多个：

```mermaid
graph TB
    subgraph org["你的组织"]
        RA[研究员智能体] <-->|A2A| CA[编码智能体]
        RA -->|MCP| SS[搜索服务器]
        CA -->|MCP| GS[GitHub 服务器]
        AUDIT["所有智能体响应携带<br/>ACP TrajectoryMetadata"]
    end

    subgraph ext["外部（通过 ANP 进行 DID 验证）"]
        EA[外部智能体]
        PA[合作伙伴智能体]
    end

    RA <-->|ANP + A2A| EA
    CA <-->|ANP + A2A| PA

    style org fill:#f8fafc,stroke:#334155
    style ext fill:#fef2f2,stroke:#991b1b
    style AUDIT fill:#fef3c7,stroke:#d97706
```

- **MCP** 将每个智能体连接到它的工具
- **A2A** 处理智能体间的协作（内部和外部）
- **ACP** 用轨迹元数据包装响应以实现可审计性
- **ANP** 为你无法控制的智能体提供身份验证

## 动手实现

### 步骤 1：核心消息类型

每个多智能体系统都从消息格式开始。我们定义映射到真实协议使用的类型：

```typescript
import crypto from "node:crypto";

type MessageRole = "user" | "agent";

type MessagePart =
  | { kind: "text"; text: string }
  | { kind: "data"; data: unknown; mediaType: string }
  | { kind: "file"; name: string; url: string; mediaType: string };

type TrajectoryEntry = {
  reasoning: string;
  toolName?: string;
  toolInput?: unknown;
  toolOutput?: unknown;
  timestamp: number;
};

type AgentMessage = {
  id: string;
  role: MessageRole;
  parts: MessagePart[];
  trajectory?: TrajectoryEntry[];
  replyTo?: string;
  timestamp: number;
};

function createMessage(
  role: MessageRole,
  parts: MessagePart[],
  replyTo?: string
): AgentMessage {
  return {
    id: crypto.randomUUID(),
    role,
    parts,
    replyTo,
    timestamp: Date.now(),
  };
}

function textMessage(role: MessageRole, text: string): AgentMessage {
  return createMessage(role, [{ kind: "text", text }]);
}
```

注意：`MessagePart` 是多模态的（文本、结构化数据、文件），就像真实的 A2A 和 ACP 规范一样。`TrajectoryEntry` 捕获推理链，匹配 ACP 的 TrajectoryMetadata。

### 步骤 2：A2A Agent Card 和注册表

构建匹配真实 A2A 规范的智能体发现：

```typescript
type Skill = {
  id: string;
  name: string;
  description: string;
  tags: string[];
  inputModes: string[];
  outputModes: string[];
};

type AgentCard = {
  name: string;
  description: string;
  version: string;
  url: string;
  capabilities: {
    streaming: boolean;
    pushNotifications: boolean;
  };
  defaultInputModes: string[];
  defaultOutputModes: string[];
  skills: Skill[];
};

class AgentRegistry {
  private cards: Map<string, AgentCard> = new Map();

  register(card: AgentCard) {
    this.cards.set(card.name, card);
  }

  discoverBySkillTag(tag: string): AgentCard[] {
    return [...this.cards.values()].filter((card) =>
      card.skills.some((skill) => skill.tags.includes(tag))
    );
  }

  discoverByInputMode(mimeType: string): AgentCard[] {
    return [...this.cards.values()].filter(
      (card) =>
        card.defaultInputModes.includes(mimeType) ||
        card.skills.some((skill) => skill.inputModes.includes(mimeType))
    );
  }

  resolve(name: string): AgentCard | undefined {
    return this.cards.get(name);
  }

  listAll(): AgentCard[] {
    return [...this.cards.values()];
  }
}
```

这比简单的名称到能力映射丰富得多。你可以按技能标签、按输入 MIME 类型或按名称发现智能体，就像真实 A2A 规范支持的那样。

### 步骤 3：A2A 任务生命周期

构建完整的任务状态机：

```typescript
type TaskState =
  | "submitted"
  | "working"
  | "input-required"
  | "auth-required"
  | "completed"
  | "failed"
  | "canceled"
  | "rejected";

const TERMINAL_STATES: TaskState[] = [
  "completed",
  "failed",
  "canceled",
  "rejected",
];

type TaskStatus = {
  state: TaskState;
  message?: AgentMessage;
  timestamp: number;
};

type Artifact = {
  id: string;
  name: string;
  parts: MessagePart[];
};

type Task = {
  id: string;
  contextId: string;
  status: TaskStatus;
  artifacts: Artifact[];
  history: AgentMessage[];
};

type TaskEvent =
  | { kind: "statusUpdate"; taskId: string; status: TaskStatus }
  | {
      kind: "artifactUpdate";
      taskId: string;
      artifact: Artifact;
      append: boolean;
      lastChunk: boolean;
    };

type TaskHandler = (
  task: Task,
  message: AgentMessage
) => AsyncGenerator<TaskEvent>;

class TaskManager {
  private tasks: Map<string, Task> = new Map();
  private handlers: Map<string, TaskHandler> = new Map();
  private listeners: Map<string, ((event: TaskEvent) => void)[]> = new Map();

  registerHandler(agentName: string, handler: TaskHandler) {
    this.handlers.set(agentName, handler);
  }

  subscribe(taskId: string, listener: (event: TaskEvent) => void) {
    const existing = this.listeners.get(taskId) ?? [];
    existing.push(listener);
    this.listeners.set(taskId, existing);
  }

  async sendMessage(
    agentName: string,
    message: AgentMessage,
    contextId?: string
  ): Promise<Task> {
    const handler = this.handlers.get(agentName);
    if (!handler) {
      const task = this.createTask(contextId);
      task.status = {
        state: "rejected",
        timestamp: Date.now(),
        message: textMessage("agent", `No handler for ${agentName}`),
      };
      return task;
    }

    const task = this.createTask(contextId);
    task.history.push(message);
    task.status = { state: "submitted", timestamp: Date.now() };

    this.processTask(task, handler, message).catch((err) => {
      task.status = {
        state: "failed",
        timestamp: Date.now(),
        message: textMessage("agent", String(err)),
      };
    });
    return task;
  }

  getTask(taskId: string): Task | undefined {
    return this.tasks.get(taskId);
  }

  cancelTask(taskId: string): boolean {
    const task = this.tasks.get(taskId);
    if (!task || TERMINAL_STATES.includes(task.status.state)) return false;
    task.status = { state: "canceled", timestamp: Date.now() };
    this.emit(taskId, {
      kind: "statusUpdate",
      taskId,
      status: task.status,
    });
    return true;
  }

  private createTask(contextId?: string): Task {
    const task: Task = {
      id: crypto.randomUUID(),
      contextId: contextId ?? crypto.randomUUID(),
      status: { state: "submitted", timestamp: Date.now() },
      artifacts: [],
      history: [],
    };
    this.tasks.set(task.id, task);
    return task;
  }

  private async processTask(
    task: Task,
    handler: TaskHandler,
    message: AgentMessage
  ) {
    task.status = { state: "working", timestamp: Date.now() };
    this.emit(task.id, {
      kind: "statusUpdate",
      taskId: task.id,
      status: task.status,
    });

    try {
      for await (const event of handler(task, message)) {
        if (TERMINAL_STATES.includes(task.status.state)) break;

        if (event.kind === "statusUpdate") {
          task.status = event.status;
        }
        if (event.kind === "artifactUpdate") {
          const existing = task.artifacts.find(
            (a) => a.id === event.artifact.id
          );
          if (existing && event.append) {
            existing.parts.push(...event.artifact.parts);
          } else {
            task.artifacts.push(event.artifact);
          }
        }
        this.emit(task.id, event);
      }
    } catch (err) {
      task.status = {
        state: "failed",
        timestamp: Date.now(),
        message: textMessage("agent", String(err)),
      };
      this.emit(task.id, {
        kind: "statusUpdate",
        taskId: task.id,
        status: task.status,
      });
    }
  }

  private emit(taskId: string, event: TaskEvent) {
    for (const listener of this.listeners.get(taskId) ?? []) {
      listener(event);
    }
  }
}
```

这实现了真实的 A2A 任务生命周期：submitted、working、input-required、终态。处理器是异步生成器，产出事件（状态更新和工件块），匹配 SSE 流式模型。

### 步骤 4：ACP 风格审计追踪

用轨迹跟踪包装通信：

```typescript
type AuditEntry = {
  runId: string;
  agentName: string;
  input: AgentMessage[];
  output: AgentMessage[];
  trajectory: TrajectoryEntry[];
  status: "created" | "in-progress" | "completed" | "failed" | "awaiting";
  startedAt: number;
  completedAt?: number;
  sessionId?: string;
};

class AuditableRunner {
  private log: AuditEntry[] = [];
  private handlers: Map<
    string,
    (input: AgentMessage[]) => Promise<{
      output: AgentMessage[];
      trajectory: TrajectoryEntry[];
    }>
  > = new Map();

  registerAgent(
    name: string,
    handler: (input: AgentMessage[]) => Promise<{
      output: AgentMessage[];
      trajectory: TrajectoryEntry[];
    }>
  ) {
    this.handlers.set(name, handler);
  }

  async run(
    agentName: string,
    input: AgentMessage[],
    sessionId?: string
  ): Promise<AuditEntry> {
    const entry: AuditEntry = {
      runId: crypto.randomUUID(),
      agentName,
      input: structuredClone(input),
      output: [],
      trajectory: [],
      status: "created",
      startedAt: Date.now(),
      sessionId,
    };
    this.log.push(entry);

    const handler = this.handlers.get(agentName);
    if (!handler) {
      entry.status = "failed";
      return entry;
    }

    entry.status = "in-progress";
    try {
      const result = await handler(input);
      entry.output = structuredClone(result.output);
      entry.trajectory = structuredClone(result.trajectory);
      entry.status = "completed";
      entry.completedAt = Date.now();
    } catch (err) {
      entry.status = "failed";
      entry.trajectory.push({
        reasoning: `Error: ${String(err)}`,
        timestamp: Date.now(),
      });
      entry.completedAt = Date.now();
    }
    return entry;
  }

  getFullAuditLog(): AuditEntry[] {
    return structuredClone(this.log);
  }

  getAuditLogForAgent(agentName: string): AuditEntry[] {
    return structuredClone(
      this.log.filter((e) => e.agentName === agentName)
    );
  }

  getAuditLogForSession(sessionId: string): AuditEntry[] {
    return structuredClone(
      this.log.filter((e) => e.sessionId === sessionId)
    );
  }

  getTrajectoryForRun(runId: string): TrajectoryEntry[] {
    const entry = this.log.find((e) => e.runId === runId);
    return entry ? structuredClone(entry.trajectory) : [];
  }
}
```

每次智能体执行都会产生一个完整的审计条目：输入了什么、输出了什么，以及中间的工具调用和推理步骤的完整轨迹。你可以按智能体、按会话或按单次运行查询。

### 步骤 5：ANP 风格身份验证

构建基于 DID 的身份和验证：

```typescript
type VerificationMethod = {
  id: string;
  type: string;
  controller: string;
  publicKeyDer: string;
};

type DIDDocument = {
  id: string;
  verificationMethod: VerificationMethod[];
  authentication: string[];
  keyAgreement: string[];
  humanAuthorization: string[];
  service: { id: string; type: string; serviceEndpoint: string }[];
};

type AgentIdentity = {
  did: string;
  document: DIDDocument;
  privateKey: crypto.KeyObject;
  publicKey: crypto.KeyObject;
};

class IdentityRegistry {
  private documents: Map<string, DIDDocument> = new Map();

  publish(doc: DIDDocument) {
    this.documents.set(doc.id, doc);
  }

  resolve(did: string): DIDDocument | undefined {
    return this.documents.get(did);
  }

  verify(did: string, signature: string, payload: string): boolean {
    const doc = this.documents.get(did);
    if (!doc) return false;

    const authKeyIds = doc.authentication;
    const authKeys = doc.verificationMethod.filter((vm) =>
      authKeyIds.includes(vm.id)
    );

    for (const key of authKeys) {
      const publicKey = crypto.createPublicKey({
        key: Buffer.from(key.publicKeyDer, "base64"),
        format: "der",
        type: "spki",
      });
      const isValid = crypto.verify(
        null,
        Buffer.from(payload),
        publicKey,
        Buffer.from(signature, "hex")
      );
      if (isValid) return true;
    }
    return false;
  }

  requiresHumanAuth(did: string, operationKeyId: string): boolean {
    const doc = this.documents.get(did);
    if (!doc) return false;
    return doc.humanAuthorization.includes(operationKeyId);
  }
}

function createIdentity(domain: string, agentName: string): AgentIdentity {
  const did = `did:wba:${domain}:agent:${agentName}`;
  const { publicKey, privateKey } = crypto.generateKeyPairSync("ed25519");

  const publicKeyDer = publicKey
    .export({ format: "der", type: "spki" })
    .toString("base64");

  const keyId = `${did}#key-1`;
  const encKeyId = `${did}#key-x25519-1`;

  const document: DIDDocument = {
    id: did,
    verificationMethod: [
      {
        id: keyId,
        type: "Ed25519VerificationKey2020",
        controller: did,
        publicKeyDer,
      },
      {
        id: encKeyId,
        type: "X25519KeyAgreementKey2019",
        controller: did,
        publicKeyDer,
      },
    ],
    authentication: [keyId],
    keyAgreement: [encKeyId],
    humanAuthorization: [],
    service: [
      {
        id: `${did}#agent-description`,
        type: "AgentDescription",
        serviceEndpoint: `https://${domain}/agents/${agentName}/ad.json`,
      },
    ],
  };

  return { did, document, privateKey, publicKey };
}

function signPayload(identity: AgentIdentity, payload: string): string {
  return crypto
    .sign(null, Buffer.from(payload), identity.privateKey)
    .toString("hex");
}
```

这反映了真实的 ANP 身份模型：智能体拥有 DID 文档，其中包含分开的认证、密钥协商和人类授权密钥。`IdentityRegistry` 模拟了 DID 解析（在生产中这将是向智能体域名发起的 HTTP 请求）。

### 步骤 6：协议网关

将四个协议连接成一个统一的系统：

```mermaid
graph LR
    REQ[传入请求] --> ANP_V{ANP: 验证 DID}
    ANP_V -->|有效| A2A_D{A2A: 发现智能体}
    ANP_V -->|无效| REJECT[拒绝]
    A2A_D -->|已找到| ACP_A[ACP: 审计运行]
    A2A_D -->|未找到| REJECT
    ACP_A --> A2A_T[A2A: 创建任务]
    A2A_T --> RESULT[任务 + 审计条目]

    style ANP_V fill:#d1fae5,stroke:#059669
    style A2A_D fill:#dbeafe,stroke:#2563eb
    style ACP_A fill:#fef3c7,stroke:#d97706
    style A2A_T fill:#dbeafe,stroke:#2563eb
```

```typescript
class ProtocolGateway {
  private registry: AgentRegistry;
  private taskManager: TaskManager;
  private auditRunner: AuditableRunner;
  private identityRegistry: IdentityRegistry;

  constructor(
    registry: AgentRegistry,
    taskManager: TaskManager,
    auditRunner: AuditableRunner,
    identityRegistry: IdentityRegistry
  ) {
    this.registry = registry;
    this.taskManager = taskManager;
    this.auditRunner = auditRunner;
    this.identityRegistry = identityRegistry;
  }

  async delegateTask(
    fromDid: string,
    signature: string,
    targetAgent: string,
    message: AgentMessage,
    sessionId?: string
  ): Promise<{ task: Task; audit: AuditEntry } | { error: string }> {
    if (!this.identityRegistry.verify(fromDid, signature, message.id)) {
      return { error: "身份验证失败" };
    }

    const card = this.registry.resolve(targetAgent);
    if (!card) {
      return { error: `智能体 ${targetAgent} 未在注册表中找到` };
    }

    const audit = await this.auditRunner.run(
      targetAgent,
      [message],
      sessionId
    );
    const task = await this.taskManager.sendMessage(targetAgent, message);

    return { task, audit };
  }

  discoverAndDelegate(
    fromDid: string,
    signature: string,
    skillTag: string,
    message: AgentMessage
  ): Promise<{ task: Task; audit: AuditEntry } | { error: string }> {
    const candidates = this.registry.discoverBySkillTag(skillTag);
    if (candidates.length === 0) {
      return Promise.resolve({
        error: `未找到具有技能标签的智能体: ${skillTag}`,
      });
    }
    return this.delegateTask(
      fromDid,
      signature,
      candidates[0].name,
      message
    );
  }
}
```

网关在一次调用中完成四件事：
1. **ANP**：通过 DID 签名验证调用者身份
2. **A2A**：发现目标智能体并检查能力
3. **ACP**：用轨迹审计包装执行
4. **A2A**：创建具有完整生命周期跟踪的任务

### 步骤 7：将所有东西连接起来

```typescript
async function protocolDemo() {
  const registry = new AgentRegistry();
  registry.register({
    name: "researcher",
    description: "Searches and summarizes findings",
    version: "1.0.0",
    url: "https://researcher.local/a2a/v1",
    capabilities: { streaming: true, pushNotifications: false },
    defaultInputModes: ["text/plain"],
    defaultOutputModes: ["text/plain", "application/json"],
    skills: [
      {
        id: "web-research",
        name: "Web Research",
        description: "Searches the web",
        tags: ["research", "search", "summarization"],
        inputModes: ["text/plain"],
        outputModes: ["application/json"],
      },
    ],
  });
  registry.register({
    name: "coder",
    description: "Writes code from specs",
    version: "1.0.0",
    url: "https://coder.local/a2a/v1",
    capabilities: { streaming: false, pushNotifications: false },
    defaultInputModes: ["text/plain", "application/json"],
    defaultOutputModes: ["text/plain"],
    skills: [
      {
        id: "code-gen",
        name: "Code Generation",
        description: "Generates code",
        tags: ["coding", "generation"],
        inputModes: ["text/plain", "application/json"],
        outputModes: ["text/plain"],
      },
    ],
  });

  const taskManager = new TaskManager();
  const auditRunner = new AuditableRunner();

  const researchTrajectory: TrajectoryEntry[] = [];

  taskManager.registerHandler(
    "researcher",
    async function* (task, message) {
      yield {
        kind: "statusUpdate" as const,
        taskId: task.id,
        status: { state: "working" as const, timestamp: Date.now() },
      };

      researchTrajectory.push({
        reasoning: "Searching for React 19 documentation",
        toolName: "web_search",
        toolInput: { query: "React 19 compiler features" },
        toolOutput: {
          results: ["react.dev/blog/react-19", "github.com/react/react"],
        },
        timestamp: Date.now(),
      });

      researchTrajectory.push({
        reasoning: "Extracting key findings from search results",
        toolName: "doc_analysis",
        toolInput: { url: "react.dev/blog/react-19" },
        toolOutput: {
          summary:
            "React 19 compiler auto-memoizes, no manual useMemo needed",
        },
        timestamp: Date.now(),
      });

      yield {
        kind: "artifactUpdate" as const,
        taskId: task.id,
        artifact: {
          id: crypto.randomUUID(),
          name: "research-results",
          parts: [
            {
              kind: "data" as const,
              data: {
                findings: [
                  "React 19 compiler auto-memoizes components",
                  "No more manual useMemo/useCallback needed",
                  "Compiler runs at build time, not runtime",
                ],
                sources: ["react.dev/blog/react-19"],
              },
              mediaType: "application/json",
            },
          ],
        },
        append: false,
        lastChunk: true,
      };

      yield {
        kind: "statusUpdate" as const,
        taskId: task.id,
        status: { state: "completed" as const, timestamp: Date.now() },
      };
    }
  );

  auditRunner.registerAgent("researcher", async () => ({
    output: [
      textMessage("agent", "React 19 compiler auto-memoizes components"),
    ],
    trajectory: researchTrajectory,
  }));

  const identityRegistry = new IdentityRegistry();

  const coderIdentity = createIdentity("coder.local", "coder");
  const researcherIdentity = createIdentity("researcher.local", "researcher");

  identityRegistry.publish(coderIdentity.document);
  identityRegistry.publish(researcherIdentity.document);

  const gateway = new ProtocolGateway(
    registry,
    taskManager,
    auditRunner,
    identityRegistry
  );

  console.log("=== 协议演示 ===\n");

  console.log("1. 智能体发现（A2A）");
  const researchAgents = registry.discoverBySkillTag("research");
  console.log(
    `   找到 ${researchAgents.length} 个智能体:`,
    researchAgents.map((a) => a.name)
  );

  console.log("\n2. 身份验证（ANP）");
  const message = textMessage("user", "Research React 19 compiler features");
  const signature = signPayload(coderIdentity, message.id);
  const verified = identityRegistry.verify(
    coderIdentity.did,
    signature,
    message.id
  );
  console.log(`   编码者 DID: ${coderIdentity.did}`);
  console.log(`   签名验证: ${verified}`);

  console.log("\n3. 任务委派（A2A + ACP + ANP）");
  const result = await gateway.delegateTask(
    coderIdentity.did,
    signature,
    "researcher",
    message,
    "session-001"
  );

  if ("error" in result) {
    console.log(`   错误: ${result.error}`);
    return;
  }

  console.log(`   任务 ID: ${result.task.id}`);
  console.log(`   任务状态: ${result.task.status.state}`);
  console.log(`   工件数: ${result.task.artifacts.length}`);

  console.log("\n4. 审计追踪（ACP）");
  console.log(`   运行 ID: ${result.audit.runId}`);
  console.log(`   状态: ${result.audit.status}`);
  console.log(`   轨迹步骤数: ${result.audit.trajectory.length}`);
  for (const step of result.audit.trajectory) {
    console.log(`     - ${step.reasoning}`);
    if (step.toolName) {
      console.log(`       工具: ${step.toolName}`);
    }
  }

  console.log("\n5. 完整审计日志");
  const fullLog = auditRunner.getFullAuditLog();
  console.log(`   总运行次数: ${fullLog.length}`);
  for (const entry of fullLog) {
    const duration = entry.completedAt
      ? `${entry.completedAt - entry.startedAt}ms`
      : "进行中";
    console.log(`   ${entry.agentName}: ${entry.status}（${duration}）`);
  }
}

protocolDemo().catch((err) => {
  console.error("协议演示失败:", err);
  process.exitCode = 1;
});
```

## 常见问题

协议解决了正常路径。以下是生产中会出问题的地方：

**Schema 漂移。** 智能体 A 发布了声称输出 `application/json` 的 Agent Card。但 JSON Schema 在版本间发生了变化。智能体 B 解析旧格式得到垃圾数据。修复：给你的技能和输出 Schema 加版本号。A2A 规范在 Agent Card 上支持 `version` 就是为此。

**状态机违规。** 智能体处理器产出一个 `completed` 事件，然后试图产出更多工件。任务已不可变。你的代码会静默丢弃更新或抛异常。修复：在产出前检查终态。上面的 `TaskManager` 通过终态后的 `break` 来强制执行。

**信任解析失败。** 智能体 A 试图验证智能体 B 的 DID，但智能体 B 的域名宕机了。DID 文档无法获取。你是开放失败（接受未验证的智能体）还是封闭失败（拒绝一切）？ANP 推荐封闭失败加上最小信任原则。

**轨迹膨胀。** ACP 轨迹日志功能强大但开销大。一个复杂智能体每次运行 200 次工具调用会产生巨大的审计条目。修复：以可配置的详细度级别记录轨迹。为合规记录工具名称和 IO，为非受监管工作负载跳过推理步骤。

**发现风暴。** 50 个智能体在启动时同时查询 `GET /agents`。修复：用 TTL 缓存 Agent Card，错开发现间隔，或使用基于推送的注册而非轮询。

## 实际应用

### 真实实现

**A2A** 最成熟。Google 的[官方规范](https://github.com/google/A2A)在 Linux Foundation 下开源。有 Python 和 TypeScript SDK。如果你的智能体需要动态发现和协作，从这里开始。

**ACP** 正在合并到 A2A。IBM 的 [BeeAI 项目](https://github.com/i-am-bee/acp)创建了 ACP 作为 REST 优先的替代方案，但轨迹元数据概念正在被 A2A 生态系统吸收。即使你使用 A2A 作为传输层，也请使用 ACP 模式（轨迹日志、运行生命周期）。

**ANP** 最具实验性。[社区仓库](https://github.com/agent-network-protocol/AgentNetworkProtocol)有 Python SDK（AgentConnect）。元协议协商概念是真正新颖的。值得为跨组织智能体部署关注。

**MCP** 已在 Phase 13 中讲解。如果你想让智能体使用工具，MCP 是标准。

### 选择正确的协议

```mermaid
graph TD
    START{智能体需要<br/>使用工具吗？}
    START -->|是| MCP_R[使用 MCP]
    START -->|否| TALK{智能体需要<br/>互相交谈吗？}
    TALK -->|否| NONE[你不需要<br/>协议]
    TALK -->|是| AUDIT{需要合规<br/>审计追踪吗？}
    AUDIT -->|是| ACP_R[A2A + ACP<br/>轨迹模式]
    AUDIT -->|否| ORG{所有智能体<br/>都在你的组织内？}
    ORG -->|是| A2A_R[A2A<br/>Agent Cards + Tasks]
    ORG -->|否| INFRA{共享<br/>基础设施？}
    INFRA -->|是| BROKER[A2A + 消息代理]
    INFRA -->|否| ANP_R[ANP + A2A<br/>DID 验证]

    style MCP_R fill:#d1fae5,stroke:#059669
    style A2A_R fill:#dbeafe,stroke:#2563eb
    style ACP_R fill:#fef3c7,stroke:#d97706
    style ANP_R fill:#f3e8ff,stroke:#7c3aed
    style BROKER fill:#e0e7ff,stroke:#4338ca
```

## 交付建议

本课产出：
- `code/main.ts` —— 四种协议模式的完整实现
- `outputs/prompt-protocol-selector.md` —— 帮助你为系统选择协议的提示词

## 练习

1. **多跳任务委派。** 扩展 `TaskManager`，使智能体处理器可以将子任务委派给其他智能体。研究员接收一个任务，将"搜索"和"总结"子任务委派给两个专家智能体，等待两者完成，然后将结果合并到自己的工件中。

2. **流式审计追踪。** 修改 `AuditableRunner` 以支持流式模式。不要等待完整结果，而是在添加轨迹条目时实时产出 `AuditEntry` 更新。使用产生审计快照的异步生成器。

3. **DID 轮换。** 为 `IdentityRegistry` 添加密钥轮换。智能体应该能够发布带有更新密钥的新 DID 文档，同时维护一个 `previousDid` 引用。验证者在宽限期内应接受当前和先前密钥的签名。

4. **协议协商。** 实现 ANP 的元协议概念。两个智能体交换带有候选格式的 `protocolNegotiation` 消息（如"我可以说 JSON-RPC"对"我偏好 REST"）。最多 3 轮后，它们就格式达成一致或超时。约定的格式决定使用哪个 `TaskManager` 或 `AuditableRunner`。

5. **限速发现。** 添加一个 `RateLimitedRegistry` 包装器，用可配置的 TTL 缓存 Agent Card 查找，并限制每个智能体每秒的发现查询次数。模拟 100 个智能体在启动时同时发现的场景，并衡量差异。

## 关键术语

| 术语 | 常见说法 | 实际含义 |
|------|---------|---------|
| MCP | "AI 工具的协议" | 用于智能体发现和使用工具的客户端-服务器协议。智能体到工具，而非智能体到智能体。 |
| A2A | "Google 的智能体协议" | Linux Foundation 下的智能体协作点对点协议。通过 Agent Card 发现，9 状态任务生命周期，通过 SSE 流式传输。支持 JSON-RPC、REST 和 gRPC 绑定。 |
| ACP | "企业智能体消息" | IBM/BeeAI 的 REST API，用于智能体运行，带 TrajectoryMetadata：每个响应携带完整的推理链和工具调用。正在合并到 A2A。 |
| ANP | "去中心化智能体身份" | 社区协议，使用 `did:wba`（DID）实现密码学身份，HPKE 实现 E2EE，AI 驱动的元协议协商实现从未见过面的智能体通信。 |
| Agent Card（智能体卡片） | "智能体的名片" | 位于 `/.well-known/agent-card.json` 的 JSON 文档，描述技能、支持的 MIME 类型、安全方案和协议绑定。 |
| DID（去中心化标识符） | "去中心化 ID" | W3C 标准，用于托管在智能体自己域名上的密码学可验证身份。ANP 使用 `did:wba` 方法。 |
| TrajectoryMetadata（轨迹元数据） | "审计收据" | ACP 的机制，将推理步骤、工具调用及其输入/输出附加到每个智能体响应。 |
| Meta-protocol（元协议） | "智能体协商如何交谈" | ANP 的方法，智能体使用自然语言动态就数据格式达成一致，然后生成代码来处理它们。 |
| Task（任务） | "一个工作单元" | A2A 的有状态对象，跟踪从提交到完成的工作。到达终态后不可变。 |

## 延伸阅读

- [Google A2A specification](https://github.com/google/A2A) — 官方规范和 SDK（v1.0.0，Linux Foundation）
- [IBM/BeeAI ACP specification](https://github.com/i-am-bee/acp) — 用于智能体运行和轨迹元数据的 OpenAPI 3.1 规范
- [Agent Network Protocol](https://github.com/agent-network-protocol/AgentNetworkProtocol) — 基于 DID 的身份、E2EE、元协议协商
- [Model Context Protocol docs](https://modelcontextprotocol.io/) — Anthropic 的 MCP 规范（Phase 13 中讲解）
- [W3C Decentralized Identifiers](https://www.w3.org/TR/did-core/) — 支撑 ANP 的身份标准
- [RFC 9180 (HPKE)](https://www.rfc-editor.org/rfc/rfc9180) — ANP 用于 E2EE 的加密方案
- [FIPA Agent Communication Language](http://www.fipa.org/specs/fipa00061/SC00061G.html) — 现代智能体协议的学术前身
