# 第22章：缓存策略与性能调优

> "缓存是计算机科学里唯一真正的『免费午餐』——唯一的问题是，大多数人吃错了姿势。" —— 老三

第19章我们聊过成本优化，其中提到 Prompt Caching 可以节省高达 90% 的成本。但那只是 Anthropic 提供的**服务端缓存**。

真正的生产级应用，需要的是**多层缓存架构**——从 Anthropic 的 Prompt Cache，到应用层的 Redis 响应缓存，再到语义缓存和 CDN。这一章，我们把缓存这件事彻底讲透。

---

## 22.1 Anthropic Prompt Caching 深度解析

### 22.1.1 Prompt Caching 是什么

Prompt Caching 是 Anthropic API 提供的服务端缓存机制。核心思想很简单：

> 如果你在多个请求之间发送**相同的前缀内容**（system prompt、文档、few-shot 示例……），API 会把这些内容缓存起来，后续请求直接复用，不再重新计算。

**效果有多夸张？**

| 场景 | 无缓存延迟 | 有缓存延迟 | 成本降低 |
|------|-----------|-----------|---------|
| 聊一本书（10万 token） | 11.5s | 2.4s (-79%) | -90% |
| Few-shot 提示（1万 token） | 1.6s | 1.1s (-31%) | -86% |
| 多轮对话（长 system prompt） | ~10s | ~2.5s (-75%) | -53% |

### 22.1.2 如何使用 Prompt Caching

核心参数：`cache_control`

```python
import anthropic

client = anthropic.Anthropic(api_key="your-api-key")

response = client.messages.create(
    model="claude-3-5-sonnet-20241022",
    max_tokens=1024,
    system=[{
        "type": "text",
        "text": "你是一个代码审查专家。以下是公司的代码规范文档：\n\n"
                "【这里是很长的代码规范文档，可能几万 token】\n\n"
                "请严格按照以上规范审查用户提交的代码。",
        "cache_control": {"type": "ephemeral"}  # ← 关键：启用缓存
    }],
    messages=[{"role": "user", "content": "请审查这段代码：def add(a,b): return a+b"}]
)

print(response.content[0].text)
```

**关键点：**

- `cache_control: {"type": "ephemeral"}` 告诉 API：这段内容我想缓存
- 缓存有效期：**5分钟**不访问自动过期，每次访问会刷新计时
- 最小缓存长度：Claude 3.5 Sonnet/Opus 是 **1024 tokens**，Haiku 是 **2048 tokens**
- 只有**前缀**可以缓存——即 `messages` 数组里**前面的消息**，后面的无法缓存

### 22.1.3 多轮对话中的缓存策略

这是最常用的场景。把 system prompt 和对话历史的前缀都缓存：

```python
import anthropic

client = anthropic.Anthropic(api_key="your-api-key")

# 模拟一个长对话，每次都把历史消息带上
conversation_history = []

def chat(user_message: str) -> str:
    """带缓存的多轮对话"""
    global conversation_history
    
    # 添加用户消息
    conversation_history.append({
        "role": "user",
        "content": user_message
    })
    
    # 构建 messages，对前面的历史消息启用缓存
    messages = []
    for i, msg in enumerate(conversation_history):
        if i < len(conversation_history) - 2:
            # 前面的消息都缓存（除了最后两条）
            messages.append({
                **msg,
                "content": [{
                    "type": "text",
                    "text": msg["content"],
                    "cache_control": {"type": "ephemeral"}
                }]
            })
        else:
            # 最后两条不缓存（省点钱，它们会频繁变化）
            messages.append(msg)
    
    response = client.messages.create(
        model="claude-3-5-sonnet-20241022",
        max_tokens=1024,
        system=[{
            "type": "text",
            "text": "你是一个耐心的编程导师，用简洁易懂的方式回答问题。",
            "cache_control": {"type": "ephemeral"}
        }],
        messages=messages
    )
    
    assistant_reply = response.content[0].text
    conversation_history.append({
        "role": "assistant",
        "content": assistant_reply
    })
    
    # 打印缓存使用情况
    if hasattr(response, 'usage'):
        print(f"Cache creation: {response.usage.cache_creation_input_tokens} tokens")
        print(f"Cache read: {response.usage.cache_read_input_tokens} tokens")
        print(f"New input: {response.usage.input_tokens} tokens")
    
    return assistant_reply

# 测试
print(chat("什么是递归？"))
print(chat("能写一个递归的例子吗？"))
print(chat("这个例子的时间复杂度是多少？"))
```

**运行效果（第二次请求开始）：**
```
Cache creation: 0 tokens      ← 不再创建缓存
Cache read: 3421 tokens       ← 命中缓存！
New input: 89 tokens          ← 只算新消息
```

### 22.1.4 定价细节（重要！）

很多人不知道 Prompt Caching 的定价结构，结果用错了反而更贵。

| 操作 | Claude 3.5 Sonnet | Claude 3 Opus | Claude 3 Haiku |
|------|-------------------|---------------|----------------|
| 普通输入 | $3 / MTok | $15 / MTok | $0.25 / MTok |
| **Cache 写入** | $3.75 / MTok (+25%) | $18.75 / MTok (+25%) | $0.30 / MTok (+25%) |
| **Cache 读取** | $0.30 / MTok (-90%) | $1.50 / MTok (-90%) | $0.03 / MTok (-90%) |

**结论：**
- 如果一段内容**只使用一次**，开启缓存反而贵 25%
- 同一段内容**使用至少 2 次**，就开始省钱了
- 使用**5次以上**，节省非常显著

**最佳实践：**

```python
def should_enable_cache(content_tokens: int, expected_reuse_count: int) -> bool:
    """
    判断是否应该启用 Prompt Caching
    
    Args:
        content_tokens: 要缓存的内容的 token 数（以千计）
        expected_reuse_count: 预计复用次数
    
    Returns:
        bool: 是否应该启用缓存
    """
    # Sonnet 定价
    normal_cost = content_tokens * 3  # $3/MTok
    cache_write_cost = content_tokens * 3.75  # $3.75/MTok
    cache_read_cost = content_tokens * 0.30   # $0.30/MTok
    
    # 总成本对比
    without_cache = normal_cost * expected_reuse_count
    with_cache = cache_write_cost + cache_read_cost * (expected_reuse_count - 1)
    
    print(f"不使用缓存总成本: ${without_cache:.4f}")
    print(f"使用缓存总成本: ${with_cache:.4f}")
    
    return with_cache < without_cache

# 示例
print(should_enable_cache(50, 2))   # False —— 2次还不够
print(should_enable_cache(50, 5))   # True  —— 5次开始省钱
print(should_enable_cache(100, 3))  # True  —— 内容越长，越早回本
```

---

## 22.2 应用层响应缓存

Anthropic 的 Prompt Caching 很强大，但它只解决了一个问题：**相同前缀的重复计算**。

现实场景中，还有大量**语义相同但文字不同**的请求，以及**完全相同的请求**（比如用户刷新页面）。这时候需要应用层缓存。

### 22.2.1 最简单的响应缓存：内存 LRU

```python
import hashlib
import json
from functools import lru_cache
from typing import Optional

class ResponseCache:
    """简单的响应缓存，使用 TTL + LRU"""
    
    def __init__(self, max_size: int = 1000, ttl_seconds: int = 300):
        self.max_size = max_size
        self.ttl_seconds = ttl_seconds
        self._cache = {}
        self._access_times = {}
    
    def _make_key(self, model: str, messages: list, temperature: float, **kwargs) -> str:
        """生成缓存 key（对请求参数做哈希）"""
        key_data = {
            "model": model,
            "messages": messages,
            "temperature": temperature,
            **kwargs
        }
        key_str = json.dumps(key_data, sort_keys=True, ensure_ascii=False)
        return hashlib.sha256(key_str.encode()).hexdigest()
    
    def get(self, model: str, messages: list, temperature: float, **kwargs) -> Optional[str]:
        """查询缓存"""
        key = self._make_key(model, messages, temperature, **kwargs)
        if key not in self._cache:
            return None
        
        # 检查 TTL
        import time
        if time.time() - self._access_times[key] > self.ttl_seconds:
            del self._cache[key]
            del self._access_times[key]
            return None
        
        # 更新访问时间
        self._access_times[key] = time.time()
        return self._cache[key]
    
    def set(self, model: str, messages: list, temperature: float, 
            response: str, **kwargs) -> None:
        """写入缓存"""
        key = self._make_key(model, messages, temperature, **kwargs)
        
        # LRU 淘汰
        if len(self._cache) >= self.max_size:
            oldest_key = min(self._access_times, key=self._access_times.get)
            del self._cache[oldest_key]
            del self._access_times[oldest_key]
        
        import time
        self._cache[key] = response
        self._access_times[key] = time.time()

# 使用
cache = ResponseCache(max_size=500, ttl_seconds=600)

def cached_chat(model: str, messages: list, temperature: float = 0.7) -> str:
    """带缓存的聊天函数"""
    cached = cache.get(model, messages, temperature)
    if cached:
        print("🎯 Cache HIT!")
        return cached
    
    print("💸 Cache MISS - calling API...")
    client = anthropic.Anthropic(api_key="your-api-key")
    response = client.messages.create(
        model=model,
        messages=messages,
        temperature=temperature,
        max_tokens=1024
    )
    result = response.content[0].text
    cache.set(model, messages, temperature, result)
    return result
```

### 22.2.2 Redis 响应缓存（生产级）

内存缓存在多进程/多机器环境下不够用。生产环境用 Redis：

```python
import redis
import json
import hashlib
import time

class RedisResponseCache:
    """基于 Redis 的响应缓存"""
    
    def __init__(self, redis_url: str = "redis://localhost:6379", 
                 default_ttl: int = 3600):
        self.redis = redis.from_url(redis_url, decode_responses=True)
        self.default_ttl = default_ttl
    
    def _make_key(self, model: str, messages: list, **params) -> str:
        key_data = {"model": model, "messages": messages, **params}
        key_str = json.dumps(key_data, sort_keys=True, ensure_ascii=False)
        key_hash = hashlib.sha256(key_str.encode()).hexdigest()
        return f"claude:cache:{key_hash}"
    
    def get(self, model: str, messages: list, **params) -> Optional[str]:
        key = self._make_key(model, messages, **params)
        cached = self.redis.get(key)
        if cached:
            # 更新访问计数
            self.redis.incr(f"{key}:hits")
            return json.loads(cached)
        return None
    
    def set(self, model: str, messages: list, response: str, 
            ttl: Optional[int] = None, **params) -> None:
        key = self._make_key(model, messages, **params)
        ttl = ttl or self.default_ttl
        self.redis.setex(key, ttl, json.dumps(response))
        self.redis.setex(f"{key}:hits", ttl, 0)
    
    def stats(self, model: str, messages: list, **params) -> dict:
        """查看缓存统计"""
        key = self._make_key(model, messages, **params)
        hits = self.redis.get(f"{key}:hits") or 0
        ttl = self.redis.ttl(key)
        return {"hits": int(hits), "ttl": ttl}

# 使用
redis_cache = RedisResponseCache(redis_url="redis://localhost:6379")

def redis_cached_chat(messages: list, model: str = "claude-3-5-sonnet-20241022") -> str:
    cached = redis_cache.get(model, messages)
    if cached:
        print(f"🎯 Redis HIT! (hits: {redis_cache.stats(model, messages)['hits']})")
        return cached
    
    print("💸 Redis MISS - calling API...")
    client = anthropic.Anthropic(api_key="your-api-key")
    response = client.messages.create(model=model, messages=messages, max_tokens=1024)
    result = response.content[0].text
    
    # 不同内容类型用不同 TTL
    ttl = 86400 if "fact" in messages[-1]["content"].lower() else 3600
    redis_cache.set(model, messages, result, ttl=ttl)
    
    return result
```

### 22.2.3 缓存失效策略

| 策略 | 适用场景 | 实现方式 |
|------|---------|---------|
| TTL（时间到期） | 事实性内容（新闻、股价） | `SETEX key 60 value` |
| 主动失效 | 知识库更新后 | `DEL pattern:*` |
| 版本号 | Prompt 模板升级 | key 里带上版本号 |
| 语义失效 | 内容更新导致旧缓存不准确 | 记录数据版本，对比版本号 |

```python
def make_cache_key_with_version(model: str, messages: list, 
                                prompt_version: str, data_version: str) -> str:
    """在缓存 key 中加入版本号，实现精准失效"""
    key_data = {
        "model": model,
        "messages": messages,
        "prompt_version": prompt_version,  # Prompt 模板版本
        "data_version": data_version       # 知识库版本
    }
    return hashlib.sha256(json.dumps(key_data).encode()).hexdigest()

# 当知识库更新时，只需要修改 data_version，旧缓存自动失效
```

---

## 22.3 语义缓存（Semantic Cache）

传统缓存有个硬伤：**必须完全相同的请求才能命中**。

但现实中，用户可能用不同的措辞问同一个问题：
- "北京的天气怎么样？"
- "今天北京天气如何？"
- "北京今天下雨吗？"

这三个问题的**语义相同**，但字符串不同，传统缓存全部 MISS。

**语义缓存**就是解决这个问题的——用向量相似度来判断「这个问题之前是不是回答过」。

### 22.3.1 语义缓存原理

```
用户提问 → 转成向量 → 在向量库里搜索相似问题 
    ↓
  找到了相似度>阈值的历史问题？
    ↓ Yes                ↓ No
  直接返回历史答案      调用 API，存入向量库
```

### 22.3.2 实现语义缓存

```python
import numpy as np
from typing import Optional, List
import json

class SemanticCache:
    """
    语义缓存：相似问题复用答案
    
    需要：
    1. Embedding 模型（用 sentence-transformers 或 OpenAI Embeddings）
    2. 向量存储（这里用简单的余弦相似度 + 内存存储做演示）
    """
    
    def __init__(self, similarity_threshold: float = 0.92, max_entries: int = 10000):
        """
        Args:
            similarity_threshold: 相似度阈值，大于此值视为「相同问题」
            max_entries: 最大缓存条目数
        """
        self.threshold = similarity_threshold
        self.max_entries = max_entries
        self.entries: List[dict] = []  # {"embedding": [], "answer": "", "question": ""}
    
    def _get_embedding(self, text: str) -> np.ndarray:
        """
        获取文本的向量表示
        
        生产环境建议：
        - 用 OpenAI Embeddings API（质量好）
        - 或用 sentence-transformers 本地推理（速度快、免费）
        """
        try:
            from sentence_transformers import SentenceTransformer
            if not hasattr(self, '_model'):
                self._model = SentenceTransformer('all-MiniLM-L6-v2')
            return self._model.encode(text)
        except ImportError:
            # fallback：用简单的字符级哈希做演示
            print("⚠️  sentence-transformers 未安装，使用简单哈希模式（不精确）")
            return np.array([hash(c) % 1000 for c in text[:100]])
    
    def _cosine_similarity(self, a: np.ndarray, b: np.ndarray) -> float:
        """计算余弦相似度"""
        dot = np.dot(a, b)
        norm_a = np.linalg.norm(a)
        norm_b = np.linalg.norm(b)
        return dot / (norm_a * norm_b + 1e-8)
    
    def get(self, question: str) -> Optional[str]:
        """查询语义缓存"""
        if not self.entries:
            return None
        
        query_embedding = self._get_embedding(question)
        
        # 遍历所有历史问题，找最相似的
        best_score = -1
        best_answer = None
        
        for entry in self.entries:
            score = self._cosine_similarity(query_embedding, entry["embedding"])
            if score > best_score:
                best_score = score
                best_answer = entry["answer"]
        
        if best_score >= self.threshold:
            print(f"🎯 Semantic Cache HIT! (similarity: {best_score:.3f})")
            print(f"   原问题: {self.entries[0]['question'][:50]}...")
            print(f"   当前问题: {question[:50]}...")
            return best_answer
        
        return None
    
    def set(self, question: str, answer: str) -> None:
        """写入语义缓存"""
        if len(self.entries) >= self.max_entries:
            # 简单 FIFO 淘汰（生产环境应该用 LRU）
            self.entries.pop(0)
        
        embedding = self._get_embedding(question)
        self.entries.append({
            "question": question,
            "answer": answer,
            "embedding": embedding
        })

# 使用
semantic_cache = SemanticCache(similarity_threshold=0.90)

def semantic_cached_qa(question: str) -> str:
    """带语义缓存的问答"""
    cached = semantic_cache.get(question)
    if cached:
        return cached
    
    # 调用 API
    client = anthropic.Anthropic(api_key="your-api-key")
    response = client.messages.create(
        model="claude-3-5-sonnet-20241022",
        messages=[{"role": "user", "content": question}],
        max_tokens=1024
    )
    answer = response.content[0].text
    
    semantic_cache.set(question, answer)
    return answer

# 测试
print(semantic_cached_qa("北京的天气怎么样？"))        # MISS -> API call
print(semantic_cached_qa("今天北京天气如何？"))        # HIT! 相似度 > 0.90
print(semantic_cached_qa("北京今天下雨吗？"))          # HIT! 
print(semantic_cached_qa("上海天气怎么样？"))          # MISS -> API call（不同城市）
```

### 22.3.3 生产级语义缓存架构

上面的实现是单机内存版。生产环境需要：

```python
# 生产级语义缓存架构（伪代码）

class ProductionSemanticCache:
    def __init__(self):
        # 向量存储：Qdrant / Pinecone / Milvus
        self.vector_db = QdrantClient(url="http://localhost:6333")
        # Embedding：OpenAI / Cohere / 本地模型
        self.embedder = OpenAIEmbeddings(api_key="...")
        # 答案存储：Redis（向量库只存向量，答案存 Redis）
        self.redis = redis.Redis()
    
    def get(self, question: str, threshold: float = 0.92) -> Optional[str]:
        # 1. 获取问题向量
        embedding = self.embedder.embed(question)
        
        # 2. 向量搜索（top-1）
        results = self.vector_db.search(
            collection="qa_cache",
            vector=embedding,
            limit=1,
            score_threshold=threshold
        )
        
        if results:
            # 3. 从 Redis 取答案
            answer_id = results[0].id
            return self.redis.get(f"answer:{answer_id}")
        
        return None
    
    def set(self, question: str, answer: str):
        # 1. 获取向量
        embedding = self.embedder.embed(question)
        
        # 2. 存入向量库
        answer_id = hashlib.md5(question.encode()).hexdigest()
        self.vector_db.upsert(
            collection="qa_cache",
            points=[{
                "id": answer_id,
                "vector": embedding,
                "payload": {"question": question[:200]}
            }]
        )
        
        # 3. 答案存 Redis
        self.redis.setex(f"answer:{answer_id}", 86400, answer)
```

---

## 22.4 多层缓存架构

把上面几种缓存组合起来，就形成了一个**多层缓存体系**：

```
用户请求
   ↓
┌─────────────────────────────────────────────┐
│ Layer 1: 应用层语义缓存（Semantic Cache）    │
│  相似问题直接返回，零延迟                     │
│  命中率目标：30-40%                          │
└─────────────────────────────────────────────┘
   ↓ MISS
┌─────────────────────────────────────────────┐
│ Layer 2: 应用层精确缓存（Redis Cache）       │
│  完全相同请求直接返回                        │
│  命中率目标：20-30%                          │
└─────────────────────────────────────────────┘
   ↓ MISS
┌─────────────────────────────────────────────┐
│ Layer 3: Anthropic Prompt Cache（服务端）   │
│  相同前缀复用，减少 API 计算                 │
│  命中率目标：50-70%（长上下文场景）           │
└─────────────────────────────────────────────┘
   ↓ MISS
┌─────────────────────────────────────────────┐
│ Layer 4: 真正调用 Anthropic API              │
└─────────────────────────────────────────────┘
```

### 22.4.1 完整实现

```python
class MultiLayerCache:
    """
    多层缓存客户端
    
    使用方式：
        client = MultiLayerCache(anthropic_key="...", redis_url="...")
        answer = client.chat(messages=[...])
    """
    
    def __init__(self, anthropic_key: str, redis_url: str = "redis://localhost:6379"):
        self.anthropic = anthropic.Anthropic(api_key=anthropic_key)
        self.redis_cache = RedisResponseCache(redis_url)
        self.semantic_cache = SemanticCache(similarity_threshold=0.91)
        self.stats = {"semantic_hit": 0, "redis_hit": 0, "prompt_cache_hit": 0, "miss": 0}
    
    def chat(self, messages: list, model: str = "claude-3-5-sonnet-20241022",
             enable_semantic: bool = True, enable_redis: bool = True,
             enable_prompt_cache: bool = True) -> str:
        """
        多层缓存聊天
        
        Args:
            messages: 对话消息列表
            model: 模型名称
            enable_semantic: 是否启用语义缓存
            enable_redis: 是否启用 Redis 缓存
            enable_prompt_cache: 是否启用 Prompt Caching（Anthropic 服务端）
        """
        user_question = messages[-1]["content"] if messages else ""
        
        # === Layer 1: 语义缓存 ===
        if enable_semantic and user_question:
            cached = self.semantic_cache.get(user_question)
            if cached:
                self.stats["semantic_hit"] += 1
                return cached
        
        # === Layer 2: Redis 精确缓存 ===
        if enable_redis:
            cached = self.redis_cache.get(model, messages)
            if cached:
                self.stats["redis_hit"] += 1
                # 同时写入语义缓存（异步，不阻塞）
                if enable_semantic:
                    self.semantic_cache.set(user_question, cached)
                return cached
        
        # === Layer 3+4: 调用 API（带 Prompt Caching）===
        self.stats["miss"] += 1
        
        create_params = {
            "model": model,
            "messages": messages,
            "max_tokens": 1024
        }
        
        # 启用 Anthropic Prompt Caching
        if enable_prompt_cache and len(messages) > 2:
            # 对前面的消息启用缓存
            cached_messages = []
            for i, msg in enumerate(messages):
                if i < len(messages) - 1:
                    cached_messages.append({
                        **msg,
                        "content": [{
                            "type": "text",
                            "text": msg["content"],
                            "cache_control": {"type": "ephemeral"}
                        }]
                    })
                else:
                    cached_messages.append(msg)
            create_params["messages"] = cached_messages
        
        response = self.anthropic.messages.create(**create_params)
        answer = response.content[0].text
        
        # 统计 Prompt Cache 命中情况
        if hasattr(response, 'usage'):
            cache_read = response.usage.cache_read_input_tokens
            if cache_read and cache_read > 0:
                self.stats["prompt_cache_hit"] += 1
        
        # 写入各层缓存
        if enable_redis:
            self.redis_cache.set(model, messages, answer)
        if enable_semantic and user_question:
            self.semantic_cache.set(user_question, answer)
        
        return answer
    
    def print_stats(self):
        """打印缓存命中统计"""
        total = sum(self.stats.values())
        if total == 0:
            print("暂无请求数据")
            return
        
        print(f"\n📊 缓存命中统计 (总请求: {total})")
        print(f"  语义缓存命中: {self.stats['semantic_hit']} ({self.stats['semantic_hit']/total*100:.1f}%)")
        print(f"  Redis 命中:  {self.stats['redis_hit']} ({self.stats['redis_hit']/total*100:.1f}%)")
        print(f"  Prompt Cache: {self.stats['prompt_cache_hit']} (API 层面)")
        print(f"  完全 MISS:   {self.stats['miss']} ({self.stats['miss']/total*100:.1f}%)")

# 使用
client = MultiLayerCache(anthropic_key="your-api-key")

# 第一次：MISS
print(client.chat([{"role": "user", "content": "Python 怎么读文件？"}]))

# 第二次（相同问题）：Redis HIT
print(client.chat([{"role": "user", "content": "Python 怎么读文件？"}]))

# 第三次（相似问题）：语义缓存 HIT
print(client.chat([{"role": "user", "content": "Python 读取文件的方法"}]))

client.print_stats()
```

---

## 22.5 性能调优实战

### 22.5.1 并发请求优化

当需要批量处理任务时，串行调用 API 太慢。用并发：

```python
import asyncio
from anthropic import AsyncAnthropic

async def batch_chat_concurrent(questions: list[str], max_concurrent: int = 5) -> list[str]:
    """
    并发处理批量问题
    
    Args:
        questions: 问题列表
        max_concurrent: 最大并发数（注意 Rate Limit！）
    """
    client = AsyncAnthropic(api_key="your-api-key")
    
    # 用 semaphore 限制并发数
    semaphore = asyncio.Semaphore(max_concurrent)
    
    async def _chat_one(question: str) -> str:
        async with semaphore:
            response = await client.messages.create(
                model="claude-3-5-sonnet-20241022",
                messages=[{"role": "user", "content": question}],
                max_tokens=1024
            )
            return response.content[0].text
    
    tasks = [_chat_one(q) for q in questions]
    return await asyncio.gather(*tasks)

# 使用
questions = [f"解释一下 Python 的 {keyword}" for keyword in 
             ["lambda", "decorator", "generator", "async/await", "context manager"]]

results = asyncio.run(batch_chat_concurrent(questions, max_concurrent=3))
for q, a in zip(questions, results):
    print(f"Q: {q}\nA: {a[:100]}...\n")
```

### 22.5.2 连接池优化

默认情况下，每次调用 `anthropic.Anthropic()` 都会创建新连接。生产环境应该复用连接：

```python
# ❌ 错误做法：每次请求都创建 client
def bad_practice(question: str) -> str:
    client = anthropic.Anthropic(api_key="your-api-key")  # 每次都新建！
    response = client.messages.create(model="...", messages=[...])
    return response.content[0].text

# ✅ 正确做法：全局复用 client
_global_client = anthropic.Anthropic(
    api_key="your-api-key",
    # 自定义 httpx 客户端，启用连接池
    http_client=httpx.Client(
        limits=httpx.Limits(max_connections=100, max_keepalive_connections=20),
        timeout=httpx.Timeout(60.0)
    )
)

def good_practice(question: str) -> str:
    response = _global_client.messages.create(model="...", messages=[...])
    return response.content[0].text
```

### 22.5.3 Token 使用监控

```python
class TokenMonitor:
    """Token 使用监控装饰器"""
    
    def __init__(self, alert_threshold_daily: int = 1_000_000):
        self.daily_usage = 0
        self.alert_threshold = alert_threshold_daily
    
    def track(self, func):
        def wrapper(*args, **kwargs):
            response = func(*args, **kwargs)
            
            if hasattr(response, 'usage'):
                input_tokens = response.usage.input_tokens
                output_tokens = response.usage.output_tokens
                cache_read = getattr(response.usage, 'cache_read_input_tokens', 0)
                
                self.daily_usage += input_tokens + output_tokens
                
                print(f"📊 Token 使用: 输入={input_tokens}, 输出={output_tokens}, "
                      f"缓存命中={cache_read}, 今日累计={self.daily_usage}")
                
                if self.daily_usage > self.alert_threshold:
                    print(f"⚠️  警告：今日 Token 使用已超过 {self.alert_threshold}！")
            
            return response
        return wrapper

# 使用
monitor = TokenMonitor(alert_threshold_daily=100_000)

@monitor.track
def tracked_chat(messages: list):
    client = anthropic.Anthropic(api_key="your-api-key")
    return client.messages.create(
        model="claude-3-5-sonnet-20241022",
        messages=messages,
        max_tokens=1024
    )
```

---

## 22.6 本章总结

| 缓存层 | 命中条件 | 延迟 | 成本节省 | 实现难度 |
|--------|---------|------|---------|---------|
| **语义缓存** | 相似问题 | ~10ms | 100% | ⭐⭐⭐ |
| **Redis 缓存** | 完全相同请求 | ~5ms | 100% | ⭐⭐ |
| **Prompt Caching** | 相同前缀 | 降低 50-80% | 50-90% | ⭐ |
| **无缓存** | - | 基准 | 0% | - |

**最佳实践 Checklist：**

- ✅ 长 system prompt 一定开启 `cache_control`
- ✅ 多轮对话对历史消息启用缓存
- ✅ 相同内容预计复用 ≥3 次才开启缓存（否则反而贵）
- ✅ 生产环境一定加 Redis 缓存层
- ✅ FAQ / 知识库场景加语义缓存层
- ✅ 监控 `cache_read_input_tokens` 确认缓存生效
- ✅ 设置合理的 TTL，避免返回过期答案

---

**下一章预告：** 第23章《企业级应用场景》——看看真实公司是怎么用 Claude Code SDK 的，以及他们踩过的坑。

> 练习 22.1：为一个客服机器人设计完整的多层缓存方案，要求：支持 100 QPS，缓存命中率 > 70%，知识库每天更新一次。写代码实现并测试命中率。

> 练习 22.2：用 `@track` 装饰器改造你的所有 API 调用，把 Token 使用数据写入 Prometheus，用 Grafana 画一个实时仪表盘。
