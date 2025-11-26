### 📚 RustLangGraph 配置文件规范

**核心设计哲学：**
1.  **资源与逻辑分离**：LLM 的 API Key 和 URL 配置在 `resources` 中，业务逻辑在 `nodes` 中，通过 ID 引用。
2.  **数据胶水层 (`input_map`)**：不再硬编码输入变量名，而是使用 JMESPath 表达式从全局状态 (`$state`) 或上游节点 (`$input`) 提取数据。
3.  **Rust 友好**：利用 `type` 字段进行枚举分发，完美契合 `#[serde(tag = "type")]`。

#### 1. 顶层结构概览

```jsonc
{
  "meta": { ... },       // 版本、超时、最大步数
  "resources": { ... },  // [新增] 模型预设注册表 (OpenAI, Local, DB)
  "state_schema": { ... }, // 全局黑板的类型定义
  "nodes": [ ... ],      // 业务节点 (引用 Resources)
  "edges": [ ... ]       // 路由逻辑 (普通/条件/分支)
}
```

---

#### 2. 完整实战示例 (JSONC)

**场景：** 多模态智能报表助手
**流程：** 语音流输入 -> 语音转文本 -> 意图路由 -> (并行: RAG搜索+Echarts / 3D建模) -> 人工审核 -> 生成 PDF。

```jsonc
{
  // ==========================================
  // 1. 元数据 (Meta)
  // ==========================================
  "meta": {
    "id": "enterprise_report_agent_v2",
    "version": "2.0.0",
    "execution_mode": "async", // async | stream
    "max_steps": 50,
    "persistence": {
      "enabled": true,
      "backend": "redis", // redis | postgres
      "key_prefix": "sess:"
    }
  },

  // ==========================================
  // 2. 资源注册表 (Resources) [NEW]
  // 解决 API Key 硬编码问题，统一管理模型参数
  // ==========================================
  "resources": {
    // 预设 1: 主力推理模型
    "llm_reasoner": {
      "type": "model.llm",
      "provider": "openai",
      "api_key_env": "OPENAI_API_KEY", // 引用环境变量
      "config": {
        "model": "gpt-4-turbo",
        "temperature": 0.1,
        "response_format": { "type": "json_object" }
      }
    },
    // 预设 2: 本地私有模型 (处理敏感数据)
    "llm_local": {
      "type": "model.llm",
      "provider": "openai_compatible",
      "base_url": "http://localhost:11434/v1",
      "api_key_env": "OLLAMA_KEY", 
      "config": {
        "model": "deepseek-coder", 
        "stop": ["<|EOT|>"]
      }
    },
    // 预设 3: 向量数据库连接
    "vec_db_main": {
      "type": "infra.vector_db",
      "provider": "qdrant",
      "url_env": "QDRANT_URL",
      "collection": "company_wiki"
    }
  },

  // ==========================================
  // 3. 全局状态 Schema (State Schema)
  // 对应 Rust 中的 HashMap<String, GraphValue>
  // ==========================================
  "state_schema": {
    "fields": [
      { "name": "user_id", "type": "string" },
      // 支持二进制流，用于处理语音/图片
      { "name": "raw_audio", "type": "bytes" }, 
      { "name": "transcript", "type": "string" },
      { "name": "intent", "type": "string" },
      // 复杂对象结构
      { "name": "report_data", "type": "object", "default": {} },
      { "name": "render_code", "type": "string" },
      { "name": "review_status", "type": "enum", "values": ["PENDING", "APPROVED", "REJECTED"] }
    ]
  },

  // ==========================================
  // 4. 节点定义 (Nodes)
  // ==========================================
  "nodes": [
    // --- 节点 A: 语音转文字 (感知层) ---
    {
      "id": "node_stt",
      "type": "perception.audio_to_text",
      "input_map": {
        // JMESPath: 从全局状态获取二进制流
        "audio_data": "$state.raw_audio",
        "lang": "'zh'" // 静态字符串需加引号
      },
      "config": {
        "model": "whisper-large-v3"
      }
    },

    // --- 节点 B: 意图识别 (推理层) ---
    {
      "id": "node_intent",
      "type": "logic.llm",
      // 引用上方定义的资源 ID
      "resource_id": "llm_reasoner", 
      "input_map": {
        // 将上一节点的输出 (transcript) 注入 Prompt
        "user_text": "$input.text"
      },
      "config": {
        "system_prompt": "分析用户意图，返回 JSON: {\"intent\": \"REPORT\" | \"MODEL\"}"
      }
    },

    // --- 节点 C1: 向量检索 (工具层) ---
    {
      "id": "node_rag",
      "type": "knowledge.rag_search",
      "resource_id": "vec_db_main",
      "input_map": {
        "query": "$state.transcript",
        "filter": "{ user_id: $state.user_id }" // 动态构造过滤条件
      },
      "config": {
        "top_k": 5
      }
    },

    // --- 节点 C2: 生成 Echarts (生成层) ---
    {
      "id": "node_gen_chart",
      "type": "gen.ui.echarts",
      "resource_id": "llm_reasoner",
      "input_map": {
        "context": "$input.documents" // 来自 node_rag 的输出
      },
      "config": {
        "chart_type": "line"
      }
    },

    // --- 节点 D: 3D 代码生成 (本地模型) ---
    {
      "id": "node_gen_3d",
      "type": "gen.code",
      "resource_id": "llm_local", // 使用本地模型省钱/保密
      "input_map": {
        "instruction": "$state.transcript"
      },
      "config": {
        "language": "javascript",
        "framework": "three.js"
      }
    },

    // --- 节点 E: 人工审核 (控制层) ---
    {
      "id": "node_human_check",
      "type": "control.human_in_the_loop",
      "input_map": {
        "artifacts": {
          "chart": "nodes.node_gen_chart.output", // 显式引用特定节点输出
          "code": "nodes.node_gen_3d.output"
        }
      },
      "config": {
        "timeout_seconds": 3600,
        "output_to_state": "review_status" // 将审核结果回写到 state
      }
    },

    // --- 节点 F: 生成 PDF ---
    {
      "id": "node_pdf",
      "type": "doc.generate_pdf",
      "input_map": {
        "content": "$input.artifacts" // 这里接收人工审核后的 payload
      },
      "config": { "template": "report_v1.md" }
    }
  ],

  // ==========================================
  // 5. 边与路由 (Edges)
  // ==========================================
  "edges": [
    // 线性：Start -> STT -> Intent
    { "from": "__START__", "to": "node_stt" },
    { "from": "node_stt", "to": "node_intent" },

    // 路由：基于意图分类
    {
      "from": "node_intent",
      "type": "routing.switch",
      "input_map": {
        "key": "$input.intent" // 从 LLM JSON 提取 intent 字段
      },
      "branches": {
        "REPORT": "node_rag",
        "MODEL": "node_gen_3d",
        "default": "node_rag"
      }
    },

    // 分支 1：RAG -> Chart -> Human
    { "from": "node_rag", "to": "node_gen_chart" },
    { "from": "node_gen_chart", "to": "node_human_check" },

    // 分支 2：3D -> Human
    { "from": "node_gen_3d", "to": "node_human_check" },

    // 条件回环：审核通过 -> PDF，否则 -> 重做
    {
      "from": "node_human_check",
      "type": "routing.condition",
      "condition_script": "$state.review_status == 'APPROVED'",
      "branches": {
        "true": "node_pdf",
        "false": "node_intent" // 打回重做，回到意图识别或指定节点
      }
    },

    { "from": "node_pdf", "to": "__END__" }
  ]
}
```

---

### ⚙️ Rust 实现指南 (Implementation Hints)

为了解析这个 JSON，你的 Rust 代码结构应该如下：

#### 1. 枚举定义 (Tagging)
使用 `serde` 的内部标签（Internally Tagged）特性。

```rust
#[derive(Debug, Deserialize)]
#[serde(tag = "type", rename_all = "snake_case")]
pub enum NodeConfig {
    // 逻辑类
    #[serde(rename = "logic.llm")]
    LlmNode(LlmNodeConfig),
    
    // 感知类
    #[serde(rename = "perception.audio_to_text")]
    SttNode(SttNodeConfig),
    
    // 控制类
    #[serde(rename = "control.human_in_the_loop")]
    HumanNode(HumanNodeConfig),
    
    // ... 其他
}

#[derive(Debug, Deserialize)]
pub struct LlmNodeConfig {
    pub resource_id: String, // 对应 resources key
    pub input_map: HashMap<String, String>, // JMESPath 表达式
    pub config: LlmParams,
}
```

#### 2. 数据映射 (The `input_map` Magic)
你需要一个辅助函数来处理 JSON 中定义的 JMESPath：

```rust
// 伪代码：解析 input_map
// node_input_map: { "user_text": "$input.text" }
// context: 包含 global state 和 prev_node output 的大 Json Value
pub fn resolve_inputs(
    node_input_map: &HashMap<String, String>, 
    context: &serde_json::Value
) -> Result<serde_json::Value> {
    let mut resolved = serde_json::Map::new();
    for (target_key, jmespath_expr) in node_input_map {
        let expr = jmespath::compile(jmespath_expr)?;
        let result = expr.search(context)?;
        resolved.insert(target_key.clone(), serde_json::to_value(result)?);
    }
    Ok(serde_json::Value::Object(resolved))
}
```

#### 3. 资源加载 (Resource Resolver)
在图初始化时，先读取 `resources` 部分，解析环境变量，建立连接池。

```rust
struct GraphResources {
    llm_clients: HashMap<String, Box<dyn LLMProvider>>,
    db_clients: HashMap<String, Box<dyn VectorDB>>,
}
// 节点运行时，通过 resource_id 从这里借用 client
```



