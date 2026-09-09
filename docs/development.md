# 开发计划（MVP — 6+1 Phase）

> 每个 phase 都是一个**独立可运行、可测试**的最小增量。
> 严格按照顺序执行，不要跳步。

## Phase 0: 项目骨架初始化

### 目标
建立项目基础设施，确保后续代码能跑起来。

### 具体步骤

| # | 操作 | 产出文件 | 验证方式 |
|---|------|----------|----------|
| 0.1 | 创建 `backend/` 目录结构 | backend/__init__.py + 各子目录 | `ls backend/` 看到所有目录 |
| 0.2 | 创建 `requirements.txt` | requirements.txt | `pip install -r requirements.txt` 无报错 |
| 0.3 | 创建 `.env.example` | .env.example | 内容包含所有环境变量名及示例值 |
| 0.4 | 创建 `tests/` 目录 + `__init__.py` | tests/__init__.py | pytest 能找到这个目录 |
| 0.5 | 编写简单 smoke test | tests/test_smoke.py | `pytest tests/test_smoke.py` 通过 |

### 依赖清单（预估）
```
fastapi>=0.115,<1
uvicorn>=0.30,<1
python-dotenv>=1.0,<2
chromadb>=0.5,<1
tiktoken>=0.7,<1
python-multipart>=0.0.9,<1
PyMuPDF>=1.24,<2
openai>=1.30,<2
pytest>=8.0,<9
```

### 为什么不一步到位？
Phase 0 只做基建，不写业务逻辑。这样你可以快速看到"空壳项目跑起来了"的成就感，然后再一点点填充血肉。

---

## Phase 1: 文档解析

### 目标
能正确解析 `.md`、`.txt`、`.pdf` 三种格式，输出纯文本。

### 具体步骤

| # | 操作 | 产出文件 | 验证方式 |
|---|------|----------|----------|
| 1.1 | 写 `config.py` | config.py | import 后能拿到配置对象 |
| 1.2 | 写 `parser.py`（TXT + MD） | parser.py | 单元测试通过 |
| 1.3 | 写 `parser.py`（PDF） | parser.py | 单元测试通过 |
| 1.4 | 集成测试 | tests/test_parser.py | 对真实文件跑一遍，对比预期输出 |

### config.py 设计说明

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    llm_base_url: str
    llm_model: str
    llm_api_key: str
    embed_base_url: str = "http://localhost:11434"
    embed_model: str = "nomic-embed-text"

settings = Settings()
```

**为什么用 pydantic-settings？**
- 类型自动校验
- 环境变量名直接映射
- `.env` 加载一行搞定

### parser.py 设计说明

```python
def parse_txt(path: str) -> str: ...
def parse_md(path: str) -> str: ...
def parse_pdf(path: str) -> str: ...
```

三个函数职责单一，互不耦合。

### 如何测试

```bash
# 准备 sample 文件
echo "# Hello" > /tmp/sample.md
echo "Hello World" > /tmp/sample.txt
# （创建一个简单的 PDF 用于测试）

# 跑测试
pytest tests/test_parser.py -v
```

### 下一步
Phase 2: 文本分块（Chunking）

---

## Phase 2: 文本分块（Chunking）

### 目标
将长文本拆成大小合理的片段，保留语义连续性。

### 具体步骤

| # | 操作 | 产出文件 | 验证方式 |
|---|------|----------|----------|
| 2.1 | 写 `chunker.py` | chunker.py | 单元测试验证 chunk 数量和大小 |
| 2.2 | 端到端串联 P1+P2 | 命令行脚本或 REPL | 传入一个 PDF → 得到 chunks |

### chunker.py 设计说明

```python
@dataclass
class Chunk:
    text: str        # 分块内容
    chunk_id: str    # 唯一标识

def chunk_text(
    text: str,
    chunk_size: int = 500,
    overlap: int = 50,
) -> List[Chunk]: ...
```

**参数说明：**
- `chunk_size`: 每块最大 token 数（字符数近似）
- `overlap`: 相邻块的共享文字，避免切割掉关键上下文

### 为什么不用 LangChain 的分块器？
LangChain 的分块器功能丰富但依赖多、学习成本高。对于 MVP，手写一个简单的基于固定大小的分块器就足够了——核心概念（size + overlap）完全一样，后面想换随时可以替换。

### 如何测试

```python
# tests/test_chunker.py
def test_basic_chunking():
    long_text = "A" * 2000
    chunks = chunk_text(long_text, chunk_size=500, overlap=50)
    assert len(chunks) >= 3
    assert all(len(c.text) <= 550 for c in chunks)  # size + overlap buffer
```

### 下一步
Phase 3: Embedding + 向量存储

---

## Phase 3: Embedding + 向量存储

### 目标
把 chunked 文本向量化并存入 ChromaDB。

### 具体步骤

| # | 操作 | 产出文件 | 验证方式 |
|---|------|----------|----------|
| 3.1 | 写 `embedding/ollama_embedder.py` | ollama_embedder.py | 调 Ollama 得到 embedding 向量 |
| 3.2 | 写 `vectorstore/chroma_manager.py` | chroma_manager.py | upsert + query 回读一致 |
| 3.3 | 串联流程 | 测试脚本 | 完整链路：parse → chunk → embed → store |

### ollama_embedder.py 设计说明

```python
import httpx

async def get_embeddings(texts: list[str]) -> list[list[float]]:
    """调用 Ollama /api/embeddings 获取向量"""
    resp = await httpx.AsyncClient().post(
        f"{settings.embed_base_url}/api/embed",
        json={"model": settings.embed_model, "input": texts},
    )
    return resp.json()["embeddings"]
```

**为什么用 httpx 而不是 SDK？**
- Ollama 没有官方 Python SDK
- OpenAI 兼容的 LLM 有官方 SDK，但 Embedding 走 Ollama REST API
- 保持轻量，只加 HTTP client 这一个新依赖

### chroma_manager.py 设计说明

```python
import chromadb

class ChromaManager:
    def __init__(self, persist_directory: str = "./data/chroma"):
        self.client = chromadb.PersistentClient(path=persist_directory)

    def upsert(self, collection_name: str, chunks: List[Chunk], embeddings: list): ...
    def query(self, collection_name: str, query_embedding: list, top_k: int = 5) -> list: ...
    def delete_collection(self, collection_name: str): ...
```

**为什么封装一层？**
- 隐藏 ChromaDB 细节
- 后续如果换 Milvus/Zilliz，只改这一个文件
- 方便 Mock 做单元测试

### 如何测试

```python
# 手动验证最快
python -c "
from backend.vectorstore.chroma_manager import ChromaManager
from backend.embedding.ollama_embedder import get_embeddings
from backend.ingestion.chunker import chunk_text

# 1. 读取并分块
with open('/path/to/sample.txt') as f:
    chunks = chunk_text(f.read(), chunk_size=500, overlap=50)

# 2. 计算 embedding
embeddings = get_embeddings([c.text for c in chunks[:5]])

# 3. 存入 Chroma
mgr = ChromaManager()
mgr.upsert('test_collection', chunks[:5], embeddings)

# 4. 查一下
results = mgr.query('test_collection', embeddings[0], top_k=3)
print(results)
"
```

### 下一步
Phase 4: 检索链路

---

## Phase 4: 检索链路（Retrieval）

### 目标
用户输入问题后，能从 ChromaDB 中检索出最相关的 Top-K 个知识片段。

### 具体步骤

| # | 操作 | 产出文件 | 验证方式 |
|---|------|----------|----------|
| 4.1 | 写 `retriever.py` | retriever.py | 给定已知答案的问题，验证检索出的 chunk 是否正确 |
| 4.2 | 集成测试 | tests/test_retriever.py | 上传一个文档，问相关和无关问题各一次 |

### retriever.py 设计说明

```python
class Retriever:
    def __init__(self, chroma_mgr: ChromaManager, embedder: OllamaEmbedder):
        self.chroma = chroma_mgr
        self.embedder = embedder

    async def retrieve(self, query: str, top_k: int = 5) -> List[Chunk]:
        # 1. Query 也先做 embedding
        query_embedding = await self.embedder.get_embeddings([query])[0]
        # 2. 查 Chroma
        results = self.chroma.query("knowledge_base", query_embedding, top_k)
        # 3. 返回原文
        return [results[i]["metadata"]["text"] for i in range(len(results))]
```

### 如何测试

**人工评估法（最简单有效）：**
1. 上传一篇关于 RAG 的文章
2. 问："什么是 RAG？" —— 看检索到的 chunk 里是否包含 RAG 定义
3. 问："今天吃什么？" —— 看是否没有强相关的 chunk 返回（这很正常）

**自动化评估（进阶）：**
```python
# tests/test_retriever.py
def test_retrieval_returns_relevant_chunks():
    # 构造一个已知内容的 Collection
    # 然后检索相关问题，断言返回的内容包含关键词
    pass
```

### 下一步
Phase 5: LLM 调用 + 问答生成

---

## Phase 5: LLM 调用 + 问答生成

### 目标
把检索到的 context 和用户问题一起发给线上 LLM，获得回答。

### 具体步骤

| # | 操作 | 产出文件 | 验证方式 |
|---|------|----------|----------|
| 5.1 | 写 `llm/provider.py` | provider.py | 能用不同 BASE_URL/MODEL 成功调用 |
| 5.2 | 写 `generation/generator.py` | generator.py | Prompt template 正确拼接 |
| 5.3 | 端到端联调 | 集成测试 | 完整链路：parse → chunk → embed → store → retrieve → generate |

### provider.py 设计说明

```python
import openai
from dotenv import load_dotenv

load_dotenv()

def get_llm_client():
    """根据环境变量动态创建 OpenAI 兼容客户端"""
    from backend.config import settings
    return openai.AsyncOpenAI(
        api_key=settings.llm_api_key,
        base_url=settings.llm_base_url,
    )

async def chat(messages: list[dict], stream: bool = False):
    client = get_llm_client()
    response = await client.chat.completions.create(
        model=settings.llm_model,
        messages=messages,
        stream=stream,
    )
    if stream:
        return _stream_response(response)
    else:
        return response.choices[0].message.content
```

**为什么用 `openai` SDK 而不是 http 直发？**
- SDK 自带重试、超时处理
- 流式响应处理更方便
- 所有兼容厂商都支持相同的 API 格式

### generation/generator.py 设计说明

```python
SYSTEM_PROMPT = """\
你是一个智能知识库助手。你的任务是根据下方提供的参考资料，
回答用户的问题。

规则：
1. 只使用提供的参考资料回答问题
2. 如果资料中没有相关信息，请诚实告诉用户你不知道
3. 回答要准确、简洁、有条理
4. 可以适当引用资料的来源（文件名或章节）\
"""

def build_rag_prompt(query: str, context: list[str]) -> list[dict]:
    """组装 RAG 场景下的 prompt"""
    context_text = "\n\n---\n\n".join(context)
    return [
        {"role": "system", "content": SYSTEM_PROMPT},
        {"role": "user", "content": (
            f"以下是相关资料：\n\n{context_text}\n\n"
            f"用户问题：{query}"
        )},
    ]
```

### 如何测试

```bash
# 1. 先测 LLM 基本调用
python -c "
from backend.llm.provider import chat
answer = await chat([{'role': 'user', 'content': '你好'}])
print(answer)
"

# 2. 端到端
python -c "
# 完整链路...
from backend.main import upload_and_ask  # 假设封装了方法
result = upload_and_ask(file_path='sample.pdf', question='RAG是什么？')
print(result)
"
```

### 下一步
Phase 6: Backend REST API

---

## Phase 6: Backend REST API

### 目标
通过 HTTP 接口对外提供服务，可用 curl 或任何 HTTP 工具调用。

### 具体步骤

| # | 操作 | 产出文件 | 验证方式 |
|---|------|----------|----------|
| 6.1 | 写 `main.py` 路由 | main.py | FastAPI 启动无报错 |
| 6.2 | `POST /upload` 接口 | main.py | curl 上传文件，返回 success |
| 6.3 | `GET /ask` 接口 | main.py | curl 提问，返回 LLM 回答 |
| 6.4 | 流式输出（SSE） | main.py | 浏览器实时显示回答 |

### main.py 设计说明

```python
from fastapi import FastAPI, UploadFile, File
from fastapi.responses import StreamingResponse

app = FastAPI(title="Knowledge Agent")

@app.post("/upload")
async def upload(file: UploadFile = File(...)):
    """上传文档，解析、分块、入库"""
    # 保存临时文件 → 解析 → 分块 → embedding → 存 Chroma
    return {"status": "success", "chunks": n}

@app.get("/ask")
async def ask(question: str):
    """提问，检索知识库 + LLM 回答"""
    # 检索 → 生成 → 返回
    return {"answer": answer}

@app.get("/ask/stream")
async def ask_stream(question: str):
    """流式回答"""
    return StreamingResponse(generator(), media_type="text/event-stream")
```

### 如何测试

```bash
# 1. 启动服务
uvicorn backend.main:app --reload

# 2. 上传文件
curl -F "file=@./sample.pdf" http://localhost:8000/upload

# 3. 提问
curl "http://localhost:8000/ask?question=RAG是什么？"

# 4. 流式
curl -N "http://localhost:8000/ask/stream?question=RAG是什么？"
```

### 下一步
Phase 7: Agent 升级（智能路由）

---

## Phase 7: Agent 路由（Agent Router）

### 目标
让系统"知道什么时候需要查知识库"，区分闲聊 vs 领域问题。

### 具体步骤

| # | 操作 | 产出文件 | 验证方式 |
|---|------|----------|----------|
| 7.1 | 写 `agent/router.py` | router.py | 判断闲聊/领域问题的准确率 |
| 7.2 | 改造 ask 链路 | main.py + generator.py | 闲聊直接回复、领域走 RAG |
| 7.3 | 对比测试 | 手动测试 | 同一问题带/不带 router 的回答差异 |

### router.py 设计说明

```python
ROUTER_PROMPT = """\
判断以下用户问题是否需要查询知识库来回答。
可能的类别：
- CHAT：日常闲聊、问候、通用知识（不需要知识库）
- KNOWLEDGE：需要专业领域知识的问答（需要查知识库）

用户问题：{question}
请只回复类别名称：CHAT 或 KNOWLEDGE\
"""

async def classify_intent(question: str) -> str:
    """用小模型判断意图"""
    messages = [{"role": "user", "content": ROUTER_PROMPT.format(question=question)}]
    answer = await chat(messages)
    return answer.strip().upper()
```

### 核心概念澄清

| 维度 | RAG | Agent |
|------|-----|-------|
| 本质 | LLM + 外部知识补充 | 自主决策系统 |
| 行为模式 | 总是查知识库 | 先判断，决定是否查 |
| 能力边界 | 只能回答已有知识的范围 | 可扩展更多工具（搜索、计算等） |
| 复杂度 | 低：一条链路 | 中高：感知→决策→行动循环 |

**现阶段用 Prompt 分类就够了。** 后续如果要扩展为真正的 Agent，可以引入 ReAct 框架或多步推理。

### 下一步
前端界面（可选，最后阶段）

---

## 全局注意事项

### 测试策略
- 每个 phase 都有对应的单元测试
- 尽量用最少的测试覆盖核心逻辑
- 集成测试用真实数据跑一遍即可

### 不要做的东西（明确排除）
- ❌ 用户登录 / JWT
- ❌ 权限管理 / RBAC
- ❌ 数据库持久化（除了 ChromaDB）
- ❌ Docker / Kubernetes
- ❌ Celery / 异步任务队列
- ❌ 前端界面（Phase 8+）
- ❌ 多文档版本管理
- ❌ 复杂的错误恢复机制

### 如果需要重构
在 Phase 5~6 之后，如果发现代码结构松散，可以进行一次集中重构。重构前我会分析现有代码并提出重构方案，由你确认后再执行。
