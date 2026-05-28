# 附录 M：Claude Code SDK 与 RAG 集成实战

> 本章介绍如何将 Claude Code SDK 与 RAG（检索增强生成）技术集成，构建拥有外部知识库的智能应用。
> 
> **更新时间**：2026-05-28
> 
> **代码环境**：Python 3.10+，Node.js 18+

---

## 1. RAG 技术概述

### 1.1 为什么需要 RAG

大型语言模型（LLM）虽然具备强大的生成能力，但存在以下固有缺陷：

| 缺陷 | 说明 | RAG 解决方案 |
|------|------|-------------|
| **知识时效性差** | 训练数据有限，无法获知最新信息 | 接入实时知识库 |
| **幻觉问题** | 可能生成看似合理但错误的内容 | 基于检索到的真实文档生成 |
| **领域专业知识匮乏** | 通用模型缺乏垂直领域深度知识 | 构建行业专用知识库 |
| **数据隐私风险** | 上传敏感数据可能导致泄露 | 本地部署知识库 |

### 1.2 RAG 核心流程

RAG 系统的核心工作流程分为两个阶段：

```
┌─────────────────────────────────────────────────────────────┐
│                    索引阶段 (Indexing)                        │
│  文档 → 加载 → 切分 → 向量化 → 存储到向量数据库              │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                   检索阶段 (Retrieval)                      │
│  用户查询 → 向量化 → 相似度检索 → 获取 Top-K 文档          │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                   生成阶段 (Generation)                      │
│  检索结果 + 用户查询 → 构建增强 Prompt → 调用 LLM → 生成答案   │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. 向量数据库选型

### 2.1 主流向量数据库对比

| 数据库 | 类型 | 特点 | 适用场景 |
|--------|------|------|---------|
| **Pinecone** | 云托管 | 完全托管、易扩展、生产级 | 大规模生产环境 |
| **Chroma** | 开源 | 本地优先、易上手、轻量 | 原型开发、个人项目 |
| **Weaviate** | 开源 | 混合搜索、GraphQL | 需要语义检索的场景 |
| **Milvus** | 开源 | 高并发、云原生 | 超大规模数据 |
| **pgvector** | 开源 | Postgresql 扩展 | 已有 PostgreSQL 基础设施 |

### 2.2 选型建议

```
初学者/原型开发     → Chroma（5 分钟内快速搭建）
生产环境          → Pinecone（完全托管，免维护）
中小规模团队       → Weaviate（开源 + 自托管）
已有 PG 基础设施  → pgvector（统一数据管理）
```

---

## 3. ChromaDB 实战案例

### 3.1 安装与环境配置

```bash
# 安装 Chroma
pip install chromadb

# 安装嵌入模型（可选多种���
# OpenAI 嵌入
pip install openai

# 或者使用本地嵌入（如 BGE）
pip install flag-embedding
```

### 3.2 基础 CRUD 操作

```python
import chromadb
from chromadb.utils import embedding_functions

# 初始化客户端（持久化到本地目录）
client = chromadb.PersistentClient(path="./knowledge_base")

# 配置嵌入函数（使用 OpenAI text-embedding-3-small）
embedding_fn = embedding_functions.OpenAIEmbeddingFunction(
    api_key="sk-xxxxx",
    model_name="text-embedding-3-small"
)

# 创建或获取集合（类似于表）
collection = client.get_or_create_collection(
    name="company_docs",
    embedding_function=embedding_fn,
    metadata={"description": "公司内部文档知识库"}
)

# 添加文档
documents = [
    "Claude Code SDK 是 Anthropic 推出的 AI 编程助手。",
    "Claude Code 支持 Tool Call 功能，可扩展工具能力。",
    "RAG 技术通过检索增强生成，提高回答准确性。"
]

metadatas = [
    {"source": "intro", "page": 1},
    {"source": "tool-call", "page": 1},
    {"source": "rag-guide", "page": 1}
]

ids = ["doc1", "doc2", "doc3"]

collection.add(documents=documents, metadatas=metadatas, ids=ids)
print(f"已添加 {collection.count()} 个文档")
```

### 3.3 相似度检索

```python
# 相似度检索
query = "什么是 Claude Code SDK？"

results = collection.query(
    query_texts=[query],
    n_results=3  # 返回 Top-3 结果
)

print("检索结果：")
for i, (doc, meta) in enumerate(zip(results['documents'][0], results['metadatas'][0])):
    print(f"\n--- 结果 {i+1} ---")
    print(f"文档: {doc}")
    print(f"元数据: {meta}")
```

输出：
```
检索结果：

--- 结果 1 ---
文档: Claude Code SDK 是 Anthropic 推出的 AI 编程助手。
元数据: {'source': 'intro', 'page': 1}

--- 结果 2 ---
文档: Claude Code 支持 Tool Call 功能，可扩展工具能力。
元数据: {'source': 'tool-call', 'page': 1}
```

---

## 4. Pinecone 实战案例

### 4.1 安装与环境配置

```bash
# 安装 Pinecone 客户端
pip install pinecone-client openai
```

### 4.2 云端向量数据库操作

```python
from pinecone import Pinecone, ServerlessSpec
import openai
import os

# 设置 API 密钥
openai.api_key = os.getenv("OPENAI_OPENAI_API_KEY")
pc = Pinecone(api_key=os.getenv("PINECONE_API_KEY"))

# 创建索引（Pinecone 云端托管）
index_name = "claude-code-knowledge"

if index_name not in pc.list_indexes().names():
    pc.create_index(
        name=index_name,
        dimension=1536,  # text-embedding-3-small 输出维度
        metric="cosine",
        spec=ServerlessSpec(
            cloud="aws",
            region="us-west-2"
        )
    )
    print(f"索引 {index_name} 创建成功")
else:
    print(f"索引 {index_name} 已存在")

# 连接索引
index = pc.Index(index_name)
```

### 4.3 文档索引与检索

```python
# 文档数据
docs = [
    {
        "id": "doc1",
        "text": "Claude Code SDK 提供 subprocess 运行模式，可作为子进程调用。",
        "source": "sdk-docs"
    },
    {
        "id": "doc2", 
        "text": "CLAUDE.md 是项目级配置文件，定义项目规范和工具集。",
        "source": "config-guide"
    },
    {
        "id": "doc3",
        "text": "MCP 协议是 Anthropic 推出的模型上下文协议，支持扩展工具。",
        "source": "mcp-docs"
    }
]

# 批量生成嵌入并上传
def get_embeddings(texts):
    response = openai.embeddings.create(
        model="text-embedding-3-small",
        input=texts
    )
    return [r.embedding for r in response.data]

# 获取文本列表
texts = [d["text"] for d in docs]
embeddings = get_embeddings(texts)

# 准备向量数据
vectors = []
for i, (doc, emb) in enumerate(zip(docs, embeddings)):
    vectors.append({
        "id": doc["id"],
        "values": emb,
        "metadata": {"text": doc["text"], "source": doc["source"]}
    })

# 批量 upsert
index.upsert(vectors=vectors, namespace="knowledge")
print(f"已索引 {len(vectors)} 个向量")
```

### 4.4 查询检索

```python
# 查询
query_text = "CLAUDE.md 是什么？"

# 生成查询向量
query_embedding = get_embeddings([query_text])[0]

# 检索
results = index.query(
    vector=query_embedding,
    top_k=3,
    include_metadata=True,
    namespace="knowledge"
)

print("检索结果：")
for match in results.matches:
    print(f"- Score: {match.score:.4f}")
    print(f"  Text: {match.metadata['text']}")
    print(f"  Source: {match.metadata['source']}")
```

---

## 5. Claude Code SDK 集成

### 5.1 RAG + Claude Code SDK 架构

```python
"""
Claude Code SDK RAG 集成架构
=============================
用户查询 → RAG检索 → 增强Prompt → Claude Code → 生成答案
"""

class ClaudeCodeRAG:
    """Claude Code SDK + RAG 集成类"""
    
    def __init__(self, vector_db, embedding_model="text-embedding-3-small"):
        self.vector_db = vector_db
        self.embedding_model = embedding_model
    
    def _embed(self, text):
        """文本向量化"""
        import openai
        response = openai.embeddings.create(
            model=self.embedding_model,
            input=text
        )
        return response.data[0].embedding
    
    def _build_prompt(self, query, context_docs):
        """构建增强 Prompt"""
        context = "\n\n".join([
            f"[来源 {i+1}]: {doc}" 
            for i, doc in enumerate(context_docs)
        ])
        
        prompt = f"""你是一个专业的技术问答助手。请根据以下参考资料回答用户问题。

参考资料：
{context}

用户问题：{query}

请根据参考资料回答。如果参考资料中没有相关信息，请如实说明。
"""
        return prompt
    
    def query(self, user_query, top_k=3):
        """执行 RAG 查询"""
        # 1. 向量化用户查询
        query_vector = self._embed(user_query)
        
        # 2. 检索相关文档
        results = self.vector_db.query(
            vector=query_vector,
            top_k=top_k,
            include_metadata=True
        )
        
        # 3. 提取文档内容
        context_docs = []
        for match in results.matches:
            if hasattr(match, 'metadata') and 'text' in match.metadata:
                context_docs.append(match.metadata['text'])
            else:
                # Chroma 返回格式不同
                context_docs.append(match.get('text', ''))
        
        # 4. 构建增强 Prompt
        enhanced_prompt = self._build_prompt(user_query, context_docs)
        
        return {
            "prompt": enhanced_prompt,
            "sources": context_docs,
            "scores": [m.score for m in results.matches]
        }

# 使用示例（针对 Chroma）
# collection = client.get_or_create_collection(name="company_docs")
# rag = ClaudeCodeRAG(collection)

# result = rag.query("Claude Code SDK 如何使用？")
# print(result["prompt"])
```

### 5.2 与 Claude Code 子进程集成

```python
import subprocess
import json

def call_claude_code_with_rag(prompt: str,rag_context: dict):
    """
    将 RAG 检索结果传递给 Claude Code CLI
    """
    # 构建系统 prompt（包含 RAG 上下文）
    system_prompt = rag_context["prompt"]
    
    # 调用 Claude Code CLI
    result = subprocess.run(
        [
            "claude",
            "-p",
            "--print",
            system_prompt
        ],
        input=prompt,
        capture_output=True,
        text=True
    )
    
    return result.stdout

# 简化版调用（prompt 中直接包含上下文）
def simple_rag_query(query: str, collection) -> str:
    """
    简化版 RAG 查询，直接在 prompt 中包含上下文
    """
    # 检索
    results = collection.query(
        query_texts=[query],
        n_results=3
    )
    
    # 构建上下文
    context = "\n\n".join(results['documents'][0])
    
    # 构建完整 prompt
    full_prompt = f"""请根据以下参考资料回答问题。

参考资料：
{context}

问题：{query}

回答："""
    
    # 调用 Claude Code（或任何 LLM）
    response = subprocess.run(
        ["claude", "-p", "--print", full_prompt],
        input=query,
        capture_output=True,
        text=True
    )
    
    return response.stdout
```

### 5.3 完整示例：本地知识库问答系统

```python
"""
完整的本地知识库问答系统示例
=============================
功能：
1. 加载文档并建索引
2. 接收用户问题
3. RAG 检索
4. 构建增强 prompt
5. 调用 Claude Code 生成答案
"""

import chromadb
from chromadb.utils import embedding_functions
import subprocess
import sys

class KnowledgeBaseQA:
    """本地知识库问答系统"""
    
    def __init__(self, db_path="./kb_data"):
        self.client = chromadb.PersistentClient(path=db_path)
        self.collection = None
        self._init_collection()
    
    def _init_collection(self):
        """初始化集合"""
        embedding_fn = embedding_functions.OpenAIEmbeddingFunction(
            api_key=sys.argv[1] if len(sys.argv) > 1 else "",
            model_name="text-embedding-3-small"
        )
        self.collection = self.client.get_or_create_collection(
            name="knowledge_base",
            embedding_function=embedding_fn
        )
    
    def add_documents(self, documents: list):
        """添加文档到知识库"""
        texts = []
        metadatas = []
        ids = []
        
        for i, doc in enumerate(documents):
            texts.append(doc["text"])
            metadatas.append(doc.get("metadata", {}))
            ids.append(doc.get("id", f"doc_{i}"))
        
        self.collection.add(
            documents=texts,
            metadatas=metadatas,
            ids=ids
        )
        print(f"已添加 {len(documents)} 个文档")
    
    def query(self, question: str, top_k: int = 3) -> str:
        """问答"""
        # RAG 检索
        results = self.collection.query(
            query_texts=[question],
            n_results=top_k
        )
        
        # 构建上下文
        context = "\n\n".join([
            f"【参考{i+1}】{doc}" 
            for i, doc in enumerate(results['documents'][0])
        ])
        
        # 构建完整 prompt
        full_prompt = f"""你是一个专业的技术问答助手。根据以下知识库中的资料回答用户问题。

## 知识库内容
{context}

## 用户问题
{question}

请根据知识库内容回答。如果知识库中没有相关信息，请如实说明。

回答："""
        
        # 调用 Claude Code
        proc = subprocess.Popen(
            ["claude", "-p", "--print"],
            stdin=subprocess.PIPE,
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE,
            text=True
        )
        
        answer, _ = proc.communicate(input=full_prompt.strip(), timeout=60)
        
        return answer.strip()


# 使用示例
if __name__ == "__main__":
    # 初始化
    qa = KnowledgeBaseQA()
    
    # 添加示例文档
    qa.add_documents([
        {
            "id": "doc1",
            "text": "Claude Code SDK 是 Anthropic 推出的 AI 编程助手，支持 subprocess 模式和 HTTP API 模式。",
            "metadata": {"source": "sdk-intro"}
        },
        {
            "id": "doc2", 
            "text": "CLAUDE.md 是项目级配置文件，用于定义项目的编码规范、工具集和默认行为。",
            "metadata": {"source": "claude-md"}
        },
        {
            "id": "doc3",
            "text": "RAG（检索增强生成）通过外部知识库检索，提高 LLM 回答的准确性和可信度。",
            "metadata": {"source": "rag-guide"}
        }
    ])
    
    # 测试问答
    answer = qa.query("Claude Code SDK 有哪些使用模式？")
    print("\n=== 回答 ===\n")
    print(answer)
```

---

## 6. 进阶话题

### 6.1 文档分块策略

文档分块直接影响检索质量，常用策略：

| 策略 | 方法 | 适用场景 |
|------|------|---------|
| **固定大小** | 按字符/词数固定切分 | 通用场景 |
| **段落** | 按段落切分 | 结构化文档 |
| **滑动窗口** | 重叠切分 | 需要保持上下文 |
| **语义分块** | 按主题/意义切分 | 高质量需求 |

```python
# 固定大小分块示例
def chunk_text(text: str, chunk_size: int = 1000, overlap: int = 100):
    """固定大小分块"""
    chunks = []
    start = 0
    text_len = len(text)
    
    while start < text_len:
        end = start + chunk_size
        chunk = text[start:end]
        chunks.append(chunk)
        start += chunk_size - overlap
    
    return chunks
```

### 6.2 混合检索策略

混合检索结合向量检索和关键词检索，提高召回率：

```python
# 实现混合检索的思路
def hybrid_search(query, collection, top_k=5):
    """混合检索"""
    # 向量检索
    vector_results = collection.query(
        query_texts=[query],
        n_results=top_k * 2
    )
    
    # 可选：加入关键词过滤、重排序等
    # 返回融合结果
    return {
        "documents": vector_results['documents'][:top_k],
        "distances": vector_results['distances'][:top_k]
    }
```

### 6.3 RAG 评估指标

| 指标 | 说明 |
|------|------|
| **精确率 (Precision)** | 检索结果中相关文档的比例 |
| **召回率 (Recall)** | 相关文档被检索到的比例 |
| **MRR** | Mean Reciprocal Rank，平均倒数排名 |
| **Hit Rate** | Top-K 中至少有一条相关文档的比例 |

---

## 7. 常见问题与解决方案

### 问题 1：检索不到相关内容

**原因**：文档未正确索引或查询表达不够具体

**解决方法**：
- 检查文档是否成功添加（`collection.count()`）
- 调整 `top_k` 值或阈值
- 优化文档分块策略

### 问题 2：嵌入模型选择

**建议**：
- 生产环境：使用 OpenAI `text-embedding-3-small` 或 `text-embedding-3-large`
- 本地环境：使用 BGE-M3、qwen3-embedding 等开源模型

### 问题 3：大规模数据处理

**建议**：
- 使用 Pinecone、Milvus 等分布式向量数据库
- 采用批量 upsert 和异步处理
- 实现增量索引（只索引新增文档）

---

## 8. 小结

本章介绍了 Claude Code SDK 与 RAG 技术的集成实战：

1. **RAG 核心概念**：检索 → 增强 → 生成的流程
2. **向量数据库选型**：Chroma（原型）/Pinecone（生产）
3. **代码示例**：完整的 CRUD 和检索示例
4. **SDK 集成**：将 RAG 结果融入 Claude Code Prompt
5. **进阶话题**：分块策略、混合检索、评估指标

通过 RAG 技术，可以让 Claude Code SDK 具备外部知识库能力，显著提升在专业领域问答场景中的准确性和可信度。

---

**参考资源**：
- [Chroma 官方文档](https://docs.trychroma.com/)
- [Pinecone 官方文档](https://docs.pinecone.io/)
- [RAG 技术 2025 展望 - Agentic RAG 演进](https://segmentfault.com/a/1190000047283347)
- [2025 年向量数据库与 LLM 深度集成实践指南](https://blog.csdn.net/lxcxjxhx/article/details/152467509)