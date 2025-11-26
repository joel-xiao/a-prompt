这是一份经过全面重构的**终极版 Prompt**。

它完美融合了你最初的架构构想，并补全了我之前指出的**“数据胶水（Data Flow）”**、**“资源解耦（Resource Registry）”** 以及 **“高级记忆管理”** 等生产级系统必须具备的细节。

你可以直接复制下面的内容，发送给 AI。

***

### 🚀 RustLangGraph 架构设计与实现指令 (Master Prompt)

**角色设定：**
你是一位拥有 10 年以上经验的 Rust 系统架构师，精通异步运行时（Tokio）、编译器原理（Ownership/Lifetimes）、分布式系统设计以及大模型应用开发。
请协助我设计并开发 `RustLangGraph` —— 一个对标 `LangGraph` 的 **Rust 原生、生产级、配置驱动** 的 AI 编排库（Crate）。

**项目核心目标：**
构建一个**类型安全**但**高度动态**的图执行引擎，能够处理从简单的 RAG 对话到包含数千节点的、多模态自主智能体（Autonomous Agents）协作流程。

---

### 一、 核心架构设计 (Core Architecture)

请基于 **Trait 系统** 和 **泛型** 设计核心抽象，重点解决 Rust 静态类型系统与 AI 业务动态性之间的矛盾：

1.  **动态类型系统 (The GraphValue Enum) [至关重要]**：
    *   由于图结构是 JSON 配置驱动的，节点间传递的数据无法在编译时确定。
    *   请设计一个 `GraphValue` 枚举（类似 `serde_json::Value` 但更高效），封装 `String`, `Number`, `Boolean`, `Object`, `List`。
    *   **特性能：** 必须包含 `Bytes(Vec<u8>)` 变体以支持多模态（音频/图片流）的零拷贝传递，并实现 `From/Into` 特征以便与 `serde_json` 互操作。

2.  **数据流与映射引擎 (Data Flow & IO Mapping)**：
    *   **痛点解决**：在配置中，用户不能写 Rust 代码来传参。
    *   **解决方案**：请设计一套基于 **JMESPath** 的映射逻辑。在 Node 配置中，允许通过 `input_map` 定义输入源。
    *   **逻辑要求**：支持从 `$state`（全局状态）和 `$input`（上游节点输出）提取数据。例如：`"user_query": "$state.messages[-1].content"`。

3.  **图执行引擎 (Graph Runner)**：
    *   支持 **有向有环图 (Directed Cyclic Graph)**。
    *   **并发调度**：引擎需自动分析拓扑结构，**并行执行**无依赖的节点（例如：同时进行搜索和画图）。
    *   **生命周期管理**：妥善处理 Rust 的所有权，使用 `Arc` 和 `RwLock` (或 `DashMap`) 管理状态，避免运行时 Panic。

4.  **资源注册表 (Resource Registry) [新特性]**：
    *   将 LLM 的连接配置（API Key, Base URL）与业务逻辑节点分离。
    *   在图的配置顶层设计 `resources` 字段，节点仅通过 ID 引用模型资源。

---

### 二、 记忆、状态与上下文 (Memory, State & Context)

1.  **分层状态模型 (Layered State)**：
    *   **Blackboard (全局黑板)**：全图共享，持久化存储。
    *   **Scope (局部作用域)**：仅在特定子图或分支中传递的临时变量，执行完即销毁。

2.  **结构化消息与 Token 管理**：
    *   定义 `ChatMessage` 结构体，包含 `Role`, `Content` (Text/Image/Audio), `Usage`。
    *   **Context Window Middleware**：设计中间件，当消息历史超过 LLM 的 Token 限制时，自动执行策略（`Truncation` 滑动窗口 或 `Summarization` 摘要生成）。

---

### 三、 生产级功能补全 (Production Features)

1.  **持久化与容错 (Persistence)**：
    *   **Checkpointer Trait**：支持将图的运行状态（Snapshot）序列化保存到 Redis/Postgres。
    *   **Time Travel & Resume**：支持根据 Checkpoint ID 恢复执行，允许在“人工介入（Human-in-the-loop）”修改状态后继续运行。

2.  **多模态与流式 (Multi-modal & Streaming)**：
    *   设计适配器接口处理非文本数据（如 PDF 生成、Echarts JSON）。
    *   实现 **Event Bus**，支持将 Token 生成、工具调用等事件通过 Channel 实时流式推送到前端。

3.  **可观测性 (Observability)**：
    *   集成 `tracing` 和 `opentelemetry`，确保每个节点的输入、输出、耗时都可被追踪。

---

### 四、 交付物要求 (Deliverables)

请分两部分输出：

#### Part 1: Rust 核心代码结构
请提供核心 Structs 和 Traits 的定义，重点展示：
*   `GraphValue` 的定义及其 `Bytes` 处理。
*   `Node` Trait 以及 `async` 执行接口。
*   `GraphRunner` 如何解析 JMESPath 并调度节点。
*   `ResourceRegistry` 的加载逻辑。

#### Part 2: 完整的 JSON 配置文件示例
请编写一个覆盖以下复杂流程的 JSON 配置（展示配置设计的合理性）：
1.  **资源定义**：定义一个 OpenAI 模型和一个本地 Ollama 模型。
2.  **流程逻辑**：
    *   **输入**：用户上传语音流。
    *   **节点 A (ASR)**：将二进制语音转文字。
    *   **节点 B (Router)**：LLM 判断意图（"Report" vs "3D_Model"）。
    *   **节点 C (分支)**：如果是 Report，并行执行 `RAG_Search` 和 `Echarts_Gen`；如果是 3D，执行 `Code_Gen`。
    *   **节点 D (Human)**：人工审核生成结果（断点）。
    *   **节点 E (PDF)**：合成最终 PDF。
3.  **关键细节**：在 JSON 中必须展示如何使用 `input_map` 将 A 节点的文本传给 B，以及如何引用环境变量中的 API Key。

---

**技术栈约束：**
*   **Runtime**: `tokio`
*   **Serialization**: `serde`, `serde_json` (必须使用 `#[serde(tag = "type")]` 模式处理多态节点)
*   **Data Query**: `jmespath` crate
*   **Error Handling**: `thiserror` + `anyhow`

请开始你的设计。



