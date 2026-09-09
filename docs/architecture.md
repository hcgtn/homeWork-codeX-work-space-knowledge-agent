# 项目架构文档

## 1. 系统总览

```
┌─────────────────────────────────────────────────────────┐
│                     Frontend (Vue 3)                    │   ← Phase 8+
│              上传文件 / 提问 / 展示回答                   │
└──────────────────────────┬──────────────────────────────┘
                           │ HTTP REST API
┌──────────────────────────▼──────────────────────────────┐
│                    Backend (FastAPI)                     │
│                                                          │
│  ┌─────────────┐    ┌──────────────┐    ┌────────────┐ │
│  │ Ingestion   │    │ Retrieval    │    │ Generation │ │
│  │ 文档解析     │    │ 向量检索      │    │ LLM 生成    │ │
│  └──────┬──────┘    └──────┬───────┘    └──────┬─────┘ │
│         │                 │                   │       │
│  ┌──────▼─────────────────▼───────────────────▼────┐ │
│  │           Vector Store (ChromaDB)                │ │
│  │           Embedding Layer (Ollama)               │ │
│  └──────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

## 2. 核心组件

### 2.1 Config（配置层）
- **职责**：统一从 `.env` 加载配置，为所有模块提供参数
- **设计原则**：单点读取，全局共享，禁止硬编码
- **输出**：
  ```python
  class Settings:
      # LLM 配置
      llm_base_url: str
      llm_model: str
      llm_api_key: str

      # Embedding 配置
      embed_base_url: str = "http://localhost:11434"
      embed_model: str = "nomic-embed-text"
  ```

### 2.2 Ingestion（ ingestion 管道）
两个独立子模块，职责分离：

| 模块 | 职责 | 输入 | 输出 |
|------|------|------|------|
| `parser.py` | 提取原文 | 原始文件 (.md/.txt/.pdf) | 纯文本字符串 |
| `chunker.py` | 拆分片段 | 纯文本字符串 | List[Chunk] |

**Chunk 数据结构：**
```python
@dataclass
class Chunk:
    text: str        # 分块文本内容
    chunk_id: str    # 唯一标识（如 file_index）
```

### 2.3 Embedding（嵌入层）
- **调用 Ollama `/api/embeddings`**
- 将文本向量化，供 ChromaDB 使用
- 注意：Query Embedding 和 Document Embedding 必须使用同一个模型

### 2.4 Vector Store（向量存储）
- **ChromaDB 嵌入式运行**
- 封装标准 CRUD：upsert、query、delete、list_collections
- 每个 Collection 对应一个文档或逻辑分组

### 2.5 Retrieval（检索层）
- **流程**：问题 → Embedding → ChromaDB query → Top-K chunks
- **关键参数**：top_k（默认 5）、distance_metric

### 2.6 Generation（生成层）
- **Prompt Template**：
  ```
  <system>你是一个知识问答助手。请使用以下提供的参考资料回答问题。
  如果参考资料中没有相关信息，请诚实地告诉用户你不知道。</system>
  <context>{retrieved_chunks}</context>
  <question>{user_question}</question>
  ```
- 调用线上 LLM API（OpenAI 兼容接口），支持流式输出

### 2.7 Agent Router（路由层，Phase 7）
- 判断用户问题是否需要走 RAG 链路
- 策略：轻量分类 prompt 调 LLM 判断意图
- 决策结果：直接回复 vs 走检索 + 生成
