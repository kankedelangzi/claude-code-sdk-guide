# 附录R：Claude Code SDK 与 LlamaIndex 集成实战

> **本章目标**：学会把 LlamaIndex 的 RAG 能力与 Claude Code SDK 深度结合，用 Python 和 TypeScript 双栈实现生产级知识问答系统。

---

## R.1 为什么要整合 LlamaIndex 和 Claude Code SDK？

Claude Code SDK 擅长**推理、代码生成、工具调用**，但它本身不管理知识库。

LlamaIndex 擅长**索引、检索、RAG 管道**，但它需要强大的 LLM 作为推理引擎。

两者结合，正好互补：

```
┌─────────────┐      ┌──────────────────┐      ┌──────────────┐
│  用户提问    │─────▶│  LlamaIndex RAG  │─────▶│ Claude Code  │
│             │      │  (检索相关上下文)  │      │ SDK (推理)   │
└─────────────┘      └──────────────────┘      └──────────────┘
```

**典型场景**：
- 企业知识库问答（内部文档 + Claude 推理）
- 代码库语义搜索（搜索相关代码文件，让 Claude 解释/修改）
- 客服机器人（FAQ 检索 + Claude 生成自然回答）

---

## R.2 LlamaIndex 生态概览

LlamaIndex 有两个主要版本：

| 特性 | Python 版 (`llama-index`) | TypeScript 版 (`llamaindex`) |
|------|--------------------------|-------------------------------|
| 成熟度 | ⭐⭐⭐⭐⭐ 最全 | ⭐⭐⭐⭐ 核心功能完整 |
| 集成数量 | 300+ 连接器 | 主流向量库/LLM 覆盖 |
| 适用场景 | 数据科学、研究、后端 | 前端、全栈、Edge Runtime |
| 安装 | `pip install llama-index` | `npm install llamaindex` |

### R.2.1 安装

**Python 版（推荐用于后端 RAG 服务）**：

```bash
# 基础安装（包含常用集成）
pip install llama-index

# 最小安装（按需加集成）
pip install llama-index-core
pip install llama-index-llms-anthropic   # Anthropic LLM 集成
pip install llama-index-embeddings-openai # OpenAI Embeddings
pip install llama-index-vector-stores-chroma  # Chroma 向量库
```

**TypeScript 版（推荐用于全栈/前端项目）**：

```bash
npm install llamaindex
# 或
pnpm add llamaindex
```

---

## R.3 Python 版：用 Claude 作为 LlamaIndex 的 LLM 后端

LlamaIndex Python 版通过 `llama-index-llms-anthropic` 包原生支持 Anthropic Claude。

### R.3.1 基础配置

```python
# config.py
import os
from llama_index.core import Settings
from llama_index.llms.anthropic import Anthropic

# 设置 Anthropic API Key
os.environ["ANTHROPIC_API_KEY"] = "sk-ant-..."

# 初始化 Claude 作为默认 LLM
claude_llm = Anthropic(
    model="claude-3-5-sonnet-20241022",  # 或 claude-3-opus-20240229
    max_tokens=4096,
    temperature=0.1,
)

# 注入全局设置
Settings.llm = claude_llm
Settings.tokenizer = claude_llm.tokenizer  # 重要！避免 token 计算错误
```

> ⚠️ **注意**：Claude 3 的 tokenizer 与 GPT 不同。必须设置 `Settings.tokenizer = claude_llm.tokenizer`，否则可能触发上下文溢出错误（尤其是 200K token 模型）。

### R.3.2 构建第一个 RAG 管道

```python
# rag_basic.py
from llama_index.core import VectorStoreIndex, SimpleDirectoryReader
from llama_index.core import Settings
from llama_index.llms.anthropic import Anthropic
from llama_index.embeddings.openai import OpenAIEmbedding

# 1. 配置 LLM 和 Embedding 模型
Settings.llm = Anthropic(
    model="claude-3-5-sonnet-20241022",
    max_tokens=2048,
)
Settings.embed_model = OpenAIEmbedding(
    model="text-embedding-3-small"
)

# 2. 加载文档（支持 PDF、Word、Markdown 等）
documents = SimpleDirectoryReader(
    input_dir="./data",
    required_exts=[".md", ".txt", ".pdf"],
).load_data()

# 3. 构建向量索引
index = VectorStoreIndex.from_documents(
    documents,
    show_progress=True,
)

# 4. 创建查询引擎
query_engine = index.as_query_engine(
    similarity_top_k=3,  # 检索 top-3 相关片段
    response_mode="compact",  # 压缩模式，节省 token
)

# 5. 提问！
response = query_engine.query(
    "Claude Code SDK 的 Tool Use 功能怎么用？"
)
print(str(response))
```

**运行结果示例**：
```
Claude Code SDK 的 Tool Use 功能允许 Claude 调用外部工具。
配置步骤：
1. 定义 tools 数组，每个工具包含 name、description、input_schema
2. 在 messages 中插入 tool_use content block
3. 处理 tool_result 返回...
（引用来源：data/claude-code-docs.md，相关度 94%）
```

### R.3.3 带引用的高级查询

```python
# rag_with_citations.py
from llama_index.core.postprocessor import SentenceTransformerRerank
from llama_index.core import VectorStoreIndex, SimpleDirectoryReader
from llama_index.llms.anthropic import Anthropic

# 初始化
llm = Anthropic(model="claude-3-5-sonnet-20241022")
index = VectorStoreIndex.from_documents(
    SimpleDirectoryReader("./data").load_data()
)

# 添加 Rerank 模型（提高检索精度）
rerank = SentenceTransformerRerank(
    model="BAAI/bge-reranker-base",
    top_n=3,
)

# 创建带引用的查询引擎
query_engine = index.as_query_engine(
    similarity_top_k=10,   # 先取 10 个，rerank 后留 3 个
    node_postprocessors=[rerank],
    response_mode="refine",  # 逐节点精炼答案
    citation=True,           # 启用引用
)

response = query_engine.query("如何优化 Claude Code SDK 的响应速度？")

# 打印答案和来源
print("答案：", str(response))
print("\n来源文档：")
for node in response.source_nodes:
    print(f"  - {node.metadata.get('file_name', '未知')} "
          f"(相关度: {node.score:.2f})")
```

---

## R.4 TypeScript 版：LlamaIndex.TS 集成 Claude

LlamaIndex.TS 是 LlamaIndex 的 TypeScript 实现，支持 Node.js、Deno、Bun、Cloudflare Workers 等运行时。

### R.4.1 基础 RAG 示例

```typescript
// rag.ts
import { Anthropic } from "@llamaindex/anthropic";
import { VectorStoreIndex } from "llamaindex";
import { SimpleDirectoryReader } from "llamaindex/readers";
import { OpenAIEmbedding } from "@llamaindex/openai";

// 1. 配置 Claude 为 LLM
const llm = new Anthropic({
  model: "claude-3-5-sonnet-20241022",
  maxTokens: 2048,
  apiKey: process.env.ANTHROPIC_API_KEY,
});

// 2. 配置 Embedding 模型
const embedModel = new OpenAIEmbedding({
  model: "text-embedding-3-small",
  apiKey: process.env.OPENAI_API_KEY,
});

// 3. 加载文档
const documents = await SimpleDirectoryReader.loadData({
  directoryPath: "./data",
  requiredExts: [".md", ".txt"],
});

// 4. 构建索引
const index = await VectorStoreIndex.fromDocuments(documents, {
  embedModel,
});

// 5. 查询
const queryEngine = index.asQueryEngine({
  llm,
  similarityTopK: 3,
});

const response = await queryEngine.query({
  query: "Claude Code SDK 支持哪些编程语言？",
});

console.log(response.response);
```

### R.4.2 与 Claude Code SDK 协同工作

```typescript
// hybrid.ts - LlamaIndex 检索 + Claude Code SDK 推理
import { Anthropic } from "@llamaindex/anthropic";
import { VectorStoreIndex, SimpleDirectoryReader } from "llamaindex";
import { Anthropic as SDKAnthropic } from "@anthropic-ai/sdk";

// 第一步：LlamaIndex 检索相关上下文
async function retrieveContext(question: string): Promise<string> {
  const index = await VectorStoreIndex.fromDocuments(
    await SimpleDirectoryReader.loadData({ directoryPath: "./docs" })
  );
  const queryEngine = index.asQueryEngine({
    llm: new Anthropic({ model: "claude-3-5-sonnet-20241022" }),
  });
  const result = await queryEngine.query({ query: question });
  return result.response;
}

// 第二步：用 Claude Code SDK 深度推理
async function deepReasoning(question: string, context: string) {
  const anthropic = new SDKAnthropic({
    apiKey: process.env.ANTHROPIC_API_KEY!,
  });

  const msg = await anthropic.messages.create({
    model: "claude-3-5-sonnet-20241022",
    max_tokens: 4096,
    messages: [{
      role: "user",
      content: `根据以下参考资料回答问题：\n\n${context}\n\n问题：${question}`,
    }],
  });

  return msg.content[0].text;
}

// 主流程
async function main() {
  const question = "如何在生产环境中部署 Claude Code SDK？";
  
  console.log("🔍 检索相关文档...");
  const context = await retrieveContext(question);
  
  console.log("🧠 Claude 深度推理中...");
  const answer = await deepReasoning(question, context);
  
  console.log("\n📝 最终答案：\n", answer);
}

main();
```

**架构说明**：
- LlamaIndex 负责**检索**（速度快，专注度只需要"找相关文档"）
- Claude Code SDK 负责**推理**（能力强，处理复杂逻辑、代码生成）
- 两者通过 `context` 字符串传递信息，解耦清晰

---

## R.5 向量数据库集成

LlamaIndex 支持 20+ 向量数据库。以下是与 Claude Code SDK 结合最常用的几个。

### R.5.1 Chroma（本地开发首选）

```python
# chroma_example.py
import chromadb
from llama_index.core import VectorStoreIndex
from llama_index.vector_stores.chroma import ChromaVectorStore
from llama_index.core import StorageContext
from llama_index.llms.anthropic import Anthropic

# 1. 初始化 ChromaDB（本地持久化）
chroma_client = chromadb.PersistentClient(path="./chroma_db")
chroma_collection = chroma_client.get_or_create_collection("claude_code_docs")

# 2. 创建 LlamaIndex 向量存储
vector_store = ChromaVectorStore(chroma_collection=chroma_collection)
storage_context = StorageContext.from_defaults(vector_store=vector_store)

# 3. 构建索引（自动存入 Chroma）
index = VectorStoreIndex.from_documents(
    documents,
    storage_context=storage_context,
    llm=Anthropic(model="claude-3-5-sonnet-20241022"),
)

# 4. 后续直接使用（无需重新索引）
query_engine = index.as_query_engine()
```

### R.5.2 Pinecone（生产环境首选）

```python
# pinecone_example.py
import os
from pinecone import Pinecone
from llama_index.vector_stores.pinecone import PineconeVectorStore
from llama_index.core import VectorStoreIndex, StorageContext
from llama_index.llms.anthropic import Anthropic

# 1. 初始化 Pinecone
pc = Pinecone(api_key=os.environ["PINECONE_API_KEY"])
pinecone_index = pc.Index("claude-code-kb")

# 2. 创建向量存储
vector_store = PineconeVectorStore(
    pinecone_index=pinecone_index,
    namespace="prod-v1",  # 命名空间隔离不同版本
)

# 3. 构建索引
storage_context = StorageContext.from_defaults(vector_store=vector_store)
index = VectorStoreIndex.from_documents(
    documents,
    storage_context=storage_context,
    llm=Anthropic(model="claude-3-5-sonnet-20241022"),
)

# 4. 查询（自动从 Pinecone 检索）
query_engine = index.as_query_engine(
    similarity_top_k=5,
)
```

---

## R.6 Agent + RAG：LlamaIndex Agent 调用 Claude Code SDK

LlamaIndex 的 Agent 框架可以让 Claude 调用工具，其中一个工具就是"查询知识库"。

```python
# agent_with_rag_tool.py
from llama_index.core import VectorStoreIndex, SimpleDirectoryReader
from llama_index.core.tools import QueryEngineTool, ToolMetadata
from llama_index.agent.openai import OpenAIAgent  # 也支持 Anthropic Agent
from llama_index.llms.anthropic import Anthropic
from llama_index.core import Settings

# 1. 构建知识库查询引擎
documents = SimpleDirectoryReader("./company_docs").load_data()
index = VectorStoreIndex.from_documents(documents)
query_engine = index.as_query_engine()

# 2. 封装为 Tool
rag_tool = QueryEngineTool(
    query_engine=query_engine,
    metadata=ToolMetadata(
        name="company_knowledge_base",
        description=(
            "查询公司知识库，包含：HR 政策、技术文档、"
            "API 手册。用中文提问。"
        ),
    ),
)

# 3. 创建 Agent（使用 Claude）
agent = OpenAIAgent.from_tools(
    tools=[rag_tool],
    llm=Anthropic(model="claude-3-5-sonnet-20241022"),
    verbose=True,
)

# 4. 执行（Agent 会自动决定是否需要查知识库）
response = agent.chat(
    "我们公司的年假政策是什么？如果我有 10 天年假，可以拆分休假吗？"
)
print(response)
```

**执行流程**：
```
用户提问
  │
  ▼
Agent (Claude) 分析：需要查 HR 政策
  │
  ▼
调用 company_knowledge_base 工具
  │
  ▼
RAG 检索相关文档片段
  │
  ▼
Claude 基于检索结果生成回答
```

---

## R.7 完整实战：代码库问答系统

这个案例把"代码语义搜索"和"Claude 代码解释"结合起来。

```python
# code_qa_system.py
import os
from pathlib import Path
from llama_index.core import VectorStoreIndex, SimpleDirectoryReader
from llama_index.core import Settings
from llama_index.llms.anthropic import Anthropic
from llama_index.embeddings.openai import OpenAIEmbedding
from llama_index.core.node_parser import CodeSplitter

# 1. 配置
Settings.llm = Anthropic(
    model="claude-3-5-sonnet-20241022",
    max_tokens=4096,
)
Settings.embed_model = OpenAIEmbedding()

# 2. 加载代码文件（支持多种语言）
code_docs = SimpleDirectoryReader(
    input_dir="./my_project/src",
    required_exts=[".ts", ".tsx", ".js", ".py"],
    recursive=True,
).load_data()

# 3. 按语法分割代码（保留结构信息）
splitter = CodeSplitter(
    language="typescript",  # 或 "python"
    chunk_lines=50,
    chunk_lines_overlap=10,
    max_chars=2000,
)
nodes = splitter(code_docs)

# 4. 构建索引
index = VectorStoreIndex(nodes)

# 5. 创建查询引擎
query_engine = index.as_query_engine(
    similarity_top_k=3,
    response_mode="tree_summarize",  # 适合代码总结
)

# 6. 提问
while True:
    question = input("\n💬 问代码（输入 q 退出）: ")
    if question.lower() == 'q':
        break
    response = query_engine.query(question)
    print(f"\n🤖 Claude 回答：\n{response}\n")
    print("📚 参考代码文件：")
    for node in response.source_nodes:
        print(f"  - {node.metadata.get('file_path', '未知')}")
```

**运行示例**：
```
💬 问代码（输入 q 退出）: handleSubmit 函数做了什么？

🤖 Claude 回答：
handleSubmit 函数负责：
1. 阻止默认表单提交行为
2. 调用 validateForm() 校验输入
3. 如果校验通过，调用 API.createUser()
4. 处理错误并显示 toast 通知

📚 参考代码文件：
  - src/components/UserForm.tsx
  - src/api/users.ts
```

---

## R.8 常见问题与解决方案

### Q1: Claude 3 的 200K 上下文，还需要 RAG 吗？

**需要**。原因：
- 200K token ≈ 15 万字，但企业知识库通常远超这个量
- RAG 可以精确检索相关片段，避免无关信息干扰
- 成本：200K 全量输入比 RAG 检索贵 10-100 倍

### Q2: LlamaIndex 和 LangChain 怎么选？

| 维度 | LlamaIndex | LangChain |
|------|-----------|-----------|
| 核心优势 | RAG 检索精准 | Agent 编排灵活 |
| 学习曲线 | 平缓 | 陡峭 |
| TypeScript 支持 | ✅ 原生 | ✅ 原生 |
| 与 Claude 集成 | ✅ 原生支持 | ✅ 支持 |

**建议**：RAG 为主选 LlamaIndex，复杂 Agent 编排选 LangChain，两者也可以混用。

### Q3: tokenizer 不匹配导致报错怎么办？

```python
# ✅ 正确做法
from llama_index.llms.anthropic import Anthropic
from llama_index.core import Settings

llm = Anthropic(model="claude-3-5-sonnet-20241022")
Settings.llm = llm
Settings.tokenizer = llm.tokenizer  # 必须设置！

# ❌ 错误做法（使用默认 GPT tokenizer）
# Settings.llm = Anthropic(...)
# 不设置 tokenizer → 可能上下文溢出
```

### Q4: 如何控制 RAG 的 token 消耗？

```python
query_engine = index.as_query_engine(
    similarity_top_k=3,          # 只取最相关的 3 个片段
    response_mode="compact",      # 压缩 prompt
    max_tokens=1024,             # 限制输出长度
)
```

---

## R.9 本章小结

| 内容 | 关键点 |
|------|--------|
| Python 版集成 | `llama-index-llms-anthropic` 包，设置 `Settings.tokenizer` |
| TypeScript 版集成 | `llamaindex` + `@llamaindex/anthropic`，全栈友好 |
| 混合架构 | LlamaIndex 检索 + Claude Code SDK 推理，各司其职 |
| 向量数据库 | Chroma（开发）、Pinecone（生产） |
| Agent + RAG | 把知识库查询封装为 Tool，让 Agent 自动调用 |

**下一步**：
- 阅读 [附录M：Claude Code SDK 与 RAG 集成实战](../appendix-m-rag-integration.md) 了解底层 RAG 原理
- 阅读 [第14章：多 Agent 协作](../14-multi-agent-collaboration.md) 学习 Agent 编排

---

> **作者注**：本章代码示例基于 LlamaIndex 0.10+ 和 `@anthropic-ai/sdk` 0.39+。LlamaIndex 迭代较快，请关注官方文档获取最新 API。
