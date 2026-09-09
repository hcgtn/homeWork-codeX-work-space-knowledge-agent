# 项目指令 — knowledge-agent

## 🎯 这是什么项目？

一个**本地知识库 Agent**，核心能力是：用户上传文档，系统把文档向量化存入 ChromaDB，用户提问后检索相关片段交给 LLM 回答。最终进阶为能自主判断是否需要检索的 Agent。

---

## ⚠️ 硬性规则（违反任何一条都算错）

1. **不要擅自更换技术栈**
   - 后端必须用 FastAPI + Python
   - 前端暂不开发，最后再做
   - 向量存储必须用 ChromaDB
   - Embedding 必须用 Ollama 本地推理
   - LLM 通过 OpenAI 兼容接口调用线上 API

2. **不要在代码里写死 API Key / 密码**
   - 所有配置必须通过 `.env` + `python-dotenv` 加载
   - 参考 `docs/architecture.md` 中的 Settings 类设计

3. **每完成一个功能都要运行测试**
   - 先写对应的单元测试
   - 然后 `pytest` 验证通过后再继续下一步

4. **不要引入没有必要的依赖**
   - 能用内置库或轻量库解决就不上重型库
   - 新增依赖需要说明理由

5. **严格遵守 Phase 顺序**
   - 按 `docs/development.md` 中定义的 Phase 0 → 7 逐步推进
   - 不要跳步
   - 每个 Phase 只修改必要的文件，保持改动最小化

6. **暂时不做这些功能：**
   - ❌ 用户登录、权限管理
   - ❌ Docker / K8s / 部署
   - ❌ 复杂 Agent、多步推理
   - ❌ 数据库持久化（除 ChromaDB 外）
   - ❌ 多用户系统

---

## 📂 目录结构约定

```
knowledge-agent/
├── docs/                          # 架构和开发计划文档
│   ├── architecture.md            # 系统架构文档（必读）
│   └── development.md             # 分阶段开发计划（必读）
├── backend/                       # 后端 Python 代码
│   ├── main.py                    # FastAPI 入口
│   ├── config.py                  # 配置层
│   ├── ingestion/                 # 文档解析 + 分块
│   │   ├── parser.py
│   │   └── chunker.py
│   ├── embedding/                 # Ollama Embedding
│   │   └── ollama_embedder.py
│   ├── vectorstore/               # ChromaDB 封装
│   │   └── chroma_manager.py
│   ├── retrieval/                 # 向量检索
│   │   └── retriever.py
│   └── generation/                # Prompt + LLM 调用
│       ├── generator.py
│       └── llm/
│           └── provider.py
├── frontend/                      # Vue 3（后期再做）
├── tests/                         # 单元测试
├── data/chroma/                   # ChromaDB 数据目录
├── .env.example                   # 环境变量模板
├── requirements.txt               # Python 依赖
└── README.md                      # 项目说明
```

---

## 🔧 配置文件约定

### `.env`（本地开发使用，已加入 .gitignore）
```env
# LLM 配置（OpenAI 兼容接口）
LLM_BASE_URL=
LLM_MODEL=
LLM_API_KEY=

# Embedding 配置（Ollama 本地）
EMBED_BASE_URL=http://localhost:11434
EMBED_MODEL=nomic-embed-text
```

### `.env.example`（提交到仓库，供队友参考）
```env
# LLM 配置
LLM_BASE_URL=https://api.siliconflow.cn/v1
LLM_MODEL=deepseek-ai/DeepSeek-V3
LLM_API_KEY=your-api-key-here

# Embedding 配置
EMBED_BASE_URL=http://localhost:11434
EMBED_MODEL=nomic-embed-text
```

---

## 🏗️ LLM Provider 设计（关键抽象）

代码统一写成：
```
LLM Provider (DeepSeek / Qwen / SiliconFlow...)
    ↓
OpenAI-compatible interface (openai SDK)
    ↓
动态配置: BASE_URL + MODEL + API_KEY from .env
```

切换供应商只需要改 `.env` 中的三个值，业务代码不动。

**为什么这样设计？**
- OpenAI Chat Completions API 是行业事实标准
- 国内主流厂商（SiliconFlow、百炼、DeepSeek）都实现了兼容接口
- 一次编写，多处复用

---

## 🤖 编码规范

### Python 风格
- 函数命名用 `snake_case`
- 类命名用 `PascalCase`
- 常量用 `UPPER_SNAKE_CASE`
- 每个模块顶部不需要 docstring，直接在代码中表达意图
- 避免过深的嵌套（超过 3 层就考虑拆分）

### 错误处理
- 简单场景用 try/except + 具体异常类型
- 对外暴露的 API 要用 try/catch 包装，返回有意义的错误信息
- 不要用裸 except

### Git 提交
- 每条 commit 对应一个完整的 phase 或小功能
- commit message 格式：`[Phase X] <简短描述>`
- 示例：`[Phase 1] add document parser for md, txt, pdf`

---

## 🧪 测试约定

```bash
# 跑所有测试
pytest tests/ -v

# 跑单个测试文件
pytest tests/test_parser.py -v

# 某个 phase 完成后，至少跑一次全量测试确认无回归
pytest tests/ --tb=short
```

---

## 💡 给 Codex 的提示

当你在这个项目中工作时：

1. **先读 `docs/architecture.md` 再读 `docs/development.md`**，理解当前处于哪个 Phase
2. **每次只做当前 Phase 要求的事**，不要把后面的工作提前做了
3. **修改前先理解现有代码**，尤其是其他 module 的接口定义
4. **如果不确定某个行为是否应该做，问清楚再继续**
5. **输出要清晰**：告诉用户你做了什么、改了哪些文件、如何验证
