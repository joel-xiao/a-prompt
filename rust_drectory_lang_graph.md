这是一个基于最新架构（资源解耦、动态类型 `GraphValue`、JMESPath 映射、分层状态）重新设计的 **Rust Workspace** 项目结构。

这个结构不再是一个简单的 Demo，而是为了承载 **RustLangGraph (RLG)** 作为一个生产级 Crate 的复杂性而设计的。

### 📦 项目根目录: `rust_lang_graph`

```text
rust_lang_graph/
├── Cargo.toml                # Workspace 定义
├── Cargo.lock
├── README.md
├── examples/                 # 实战示例 (e.g., voice_to_pdf.rs)
├── docker-compose.yml        # 开发环境依赖 (Redis, Postgres, Qdrant)
│
├── crates/
│   │
│   ├── rlg_core/             # [核心层] 定义所有 Trait 和基础类型 (无业务逻辑)
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── types.rs      # 关键: GraphValue 枚举 (String, Int, Bytes, Map)
│   │       ├── node.rs       # Node Trait 定义
│   │       ├── state.rs      # Blackboard (全局) & Scope (局部) 结构
│   │       ├── memory.rs     # ChatMessage, Role, WindowManager Trait
│   │       └── error.rs      # Error 类型定义
│   │
│   ├── rlg_config/           # [配置层] 负责 JSON/YAML 序列化协议
│   │   ├── Cargo.toml        # 依赖 serde, serde_json
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── schema.rs     # 对应 JSON 规范的 Rust Structs (Tagged Enums)
│   │       ├── resources.rs  # 资源配置结构 (Provider, API Key Env)
│   │       └── validator.rs  # 配置合法性校验 (检查引用是否存在)
│   │
│   ├── rlg_engine/           # [引擎层] 核心运行时 (Runtime)
│   │   ├── Cargo.toml        # 依赖 tokio, jmespath, tracing, dashmap
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── runner.rs     # 图执行循环 (Graph Loop)
│   │       ├── scheduler.rs  # 并发调度器 (检测无依赖节点并 Spawn)
│   │       ├── mapper.rs     # 关键: JMESPath 解析器 (Input Map Resolution)
│   │       ├── registry.rs   # 资源池 (管理 HTTP Clients, DB Pools)
│   │       └── checkpointer/ # 持久化模块
│   │           ├── mod.rs
│   │           ├── redis.rs  # Redis 实现
│   │           └── sql.rs    # Postgres 实现
│   │
│   ├── rlg_nodes/            # [节点标准库] 实现具体的业务节点
│   │   ├── Cargo.toml        # 依赖 async-openai, reqwest, typst
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── perception/   # 感知层 (ASR, OCR)
│   │       ├── logic/        # 逻辑层 (LLM, Router, Branch)
│   │       ├── gen/          # 生成层 (Code, Echarts, 3D)
│   │       ├── doc/          # 文档层 (PDF, Markdown)
│   │       ├── knowledge/    # 知识层 (RAG, Vector Search)
│   │       └── control/      # 控制层 (Human-in-loop, Subgraph)
│   │
│   └── rlg_server/           # [服务层] 提供 HTTP/WebSocket 接口
│       ├── Cargo.toml        # 依赖 axum, tower-http
│       └── src/
│           ├── main.rs
│           ├── api/          # RESTful Endpoints
│           └── stream/       # SSE/WebSocket 事件流推送
```

---

### 🔍 关键模块实现细节 (Design Deep Dive)

#### 1. `rlg_core::types` (解决动态类型痛点)
这是整个系统的血液。为了让 Rust 能处理类似 Python 的动态数据流，必须定义这个枚举。

```rust
// crates/rlg_core/src/types.rs

#[derive(Clone, Debug, Serialize, Deserialize)]
#[serde(untagged)] // 兼容 serde_json
pub enum GraphValue {
    Null,
    String(String),
    Number(f64),
    Boolean(bool),
    Object(Map<String, GraphValue>),
    Array(Vec<GraphValue>),
    
    // 关键：用于多模态流的特殊类型，serde 序列化时转为 base64
    #[serde(with = "base64_serde")] 
    Bytes(Vec<u8>), 
}
```

#### 2. `rlg_engine::mapper` (解决数据映射痛点)
这是将 JSON 配置中的 `input_map` 转化为实际数据的逻辑。

```rust
// crates/rlg_engine/src/mapper.rs

pub struct DataMapper;

impl DataMapper {
    // context = Global State + Previous Node Output
    pub fn resolve(
        input_map: &HashMap<String, String>, 
        context: &GraphValue
    ) -> Result<GraphValue> {
        let mut inputs = Map::new();
        for (key, path) in input_map {
            // 使用 jmespath 库查询
            let expr = jmespath::compile(path)?;
            let result = expr.search(context)?;
            inputs.insert(key.clone(), GraphValue::from(result));
        }
        Ok(GraphValue::Object(inputs))
    }
}
```

#### 3. `rlg_engine::registry` (解决资源解耦)
管理大模型客户端连接池，避免每次请求都创建 Client。

```rust
// crates/rlg_engine/src/registry.rs

pub struct ResourceRegistry {
    // 运行时缓存：ID -> Client 实例
    llm_clients: DashMap<String, Box<dyn LLMProvider>>,
    db_clients: DashMap<String, Box<dyn VectorDB>>,
}

impl ResourceRegistry {
    pub async fn load_from_config(config: &ResourcesConfig) -> Self {
        // 1. 读取 JSON 配置
        // 2. 解析环境变量 (OPENAI_API_KEY)
        // 3. 初始化 Client 并存入 DashMap
    }
}
```

#### 4. `rlg_nodes` (标准库组织)
节点必须通过 `typetag` 或 `enum dispatch` 进行注册，以便从 JSON 动态加载。

```rust
// crates/rlg_nodes/src/lib.rs

// 使用 Enum 进行静态分发 (性能更好，且 serde 友好)
#[derive(Debug, Deserialize)]
#[serde(tag = "type")]
pub enum NodeConfig {
    #[serde(rename = "logic.llm")]
    Llm(LlmNode),
    #[serde(rename = "gen.ui.echarts")]
    Echarts(EchartsNode),
    // ...
}
```

---

### 🛠️ 推荐的 `Cargo.toml` 依赖 (Workspace Level)

在根目录的 `Cargo.toml` 中统一管理版本，确保生产环境稳定性。

```toml
[workspace]
members = ["crates/*", "examples/*"]
resolver = "2"

[workspace.dependencies]
# 核心异步
tokio = { version = "1", features = ["full"] }
futures = "0.3"
async-trait = "0.1"

# 数据处理
serde = { version = "1", features = ["derive"] }
serde_json = "1"
jmespath = "0.3" # 关键：用于 input_map
bytes = "1"
base64 = "0.21"

# Web 与 网络
axum = "0.7"
reqwest = { version = "0.11", features = ["json", "stream", "multipart"] }
async-openai = "0.18" # 官方非官方客户端，质量很高

# 数据库与缓存
redis = { version = "0.24", features = ["tokio-comp"] }
sqlx = { version = "0.7", features = ["postgres", "runtime-tokio"] }
qdrant-client = "1"

# 可观测性
tracing = "0.1"
tracing-subscriber = "0.3"
thiserror = "1"
anyhow = "1"
```

### 💡 设计总结
这个结构相比旧版的主要改进点：
1.  **Engine 与 Core 分离**：`rlg_core` 变得非常纯粹，只定义标准接口，方便第三方开发插件节点。
2.  **独立 Config Crate**：配置解析逻辑很重（涉及校验、Env 解析），单独抽离方便未来做 LSP (Language Server) 或前端可视化编辑器。
3.  **Mapper 模块化**：明确了 `DataMapper` 的职责，这是让配置驱动真正可用的核心组件。
4.  **Registry 引入**：解决了 API Key 管理和连接复用的问题，符合 12-Factor App 原则。
