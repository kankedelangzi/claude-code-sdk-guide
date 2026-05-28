# 附录J：迁移指南 —— 从其他 AI SDK 迁移到 Claude Code SDK

> "如果你已经用 OpenAI SDK 写了几千行代码，别慌，迁移没你想的那么痛。大部分改动集中在初始化和消息格式两块。" —— 老三

你已经用 OpenAI SDK 或 LangChain 搭了一套 AI 应用，现在想迁移到 Claude Code SDK（可能是因为 Claude 3.5/4 的代码能力更强，或者成本更优）。

这一章给你**三条完整迁移路径**：
1. **从 OpenAI Python SDK 迁移** → 最直接的 API 对比
2. **从 OpenAI Node.js SDK 迁移** → TypeScript 项目迁移实战
3. **从 LangChain 迁移** → 保留 LangChain 编排，换底层模型

每条路径都带**可运行的对比代码**，你可以直接照着改。

---

## J.1 为什么要从 OpenAI SDK 迁移到 Claude Code SDK？

先说清楚迁移的动力，免得你白忙活。

| 对比维度 | OpenAI SDK | Claude Code SDK (@anthropic-ai/sdk) |
|---------|-----------|--------------------------------------|
| **最强模型** | GPT-4o / o1 | Claude 3.5 Sonnet / Claude Opus 4 |
| **代码生成质量** | 好 | **更强**（HumanEval 高分） |
| **长上下文** | 128K | **200K** |
| **Tool Use 稳定性** | 好 | **更强**（少幻觉、少拒答） |
| **定价（输入）** | $5/M tokens | $3/M tokens（便宜 40%） |
| **定价（输出）** | $15/M tokens | $15/M tokens（持平） |
| **SDK 成熟度** | 非常高 | 高（2024 后快速迭代） |

**结论**：如果你的场景是**代码生成、长文档分析、复杂 Tool Use**，迁移到 Claude Code SDK 收益明显。

---

## J.2 路径一：从 OpenAI Python SDK 迁移

### J.2.1 安装依赖

```bash
# 卸载 OpenAI SDK（可选，也可以共存）
pip uninstall openai

# 安装 Claude Code SDK
pip install anthropic python-dotenv
```

### J.2.2 最基础的迁移：聊天补全

**Before（OpenAI SDK）：**

```python
import openai

client = openai.OpenAI(api_key="sk-...")

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "system", "content": "你是一个编程助手"},
        {"role": "user", "content": "用 Python 写一个快速排序"}
    ]
)

print(response.choices[0].message.content)
```

**After（Claude Code SDK）：**

```python
import anthropic

client = anthropic.Anthropic(api_key="sk-ant-...")

response = client.messages.create(
    model="claude-3-5-sonnet-20241022",
    max_tokens=1024,
    system="你是一个编程助手",
    messages=[
        {"role": "user", "content": "用 Python 写一个快速排序"}
    ]
)

print(response.content[0].text)
```

**关键差异对照表：**

| 概念 | OpenAI SDK | Claude Code SDK | 说明 |
|------|-----------|----------------|------|
| 客户端类 | `openai.OpenAI` | `anthropic.Anthropic` | 类似 |
| 聊天接口 | `chat.completions.create` | `messages.create` | 不同 |
| System Prompt | `messages` 里 `{role: "system"}` | 独立参数 `system` | **重要差异！** |
| 模型参数 | `model="gpt-4o"` | `model="claude-3-5-sonnet-..."` | 命名规则不同 |
| 最大输出长度 | `max_tokens` | `max_tokens` | 相同 |
| 回复内容提取 | `response.choices[0].message.content` | `response.content[0].text` | 结构不同 |
| 消息历史 | `messages` 数组（含 system） | `messages` 数组（不含 system） | **重要差异！** |

### J.2.3 多轮对话迁移

**Before（OpenAI SDK）：**

```python
messages = [
    {"role": "system", "content": "你是一个编程助手"},
    {"role": "user", "content": "用 Python 写快速排序"}
]

response = client.chat.completions.create(
    model="gpt-4o",
    messages=messages
)

assistant_reply = response.choices[0].message.content
messages.append({"role": "assistant", "content": assistant_reply})
messages.append({"role": "user", "content": "加上类型注解"})
```

**After（Claude Code SDK）：**

```python
system_prompt = "你是一个编程助手"
messages = [
    {"role": "user", "content": "用 Python 写快速排序"}
]

response = client.messages.create(
    model="claude-3-5-sonnet-20241022",
    max_tokens=1024,
    system=system_prompt,
    messages=messages
)

assistant_reply = response.content[0].text
messages.append({"role": "assistant", "content": assistant_reply})
messages.append({"role": "user", "content": "加上类型注解"})
```

**⚠️ 关键坑**：Claude 的 `system` 是独立参数，不要放进 `messages` 数组！如果你直接把 OpenAI 的 `messages` 数组塞给 Claude，system 消息会被当成 user 消息处理。

### J.2.4 Streaming 迁移

**Before（OpenAI SDK）：**

```python
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "写一首诗"}],
    stream=True
)

for chunk in response:
    if chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="")
```

**After（Claude Code SDK）：**

```python
with client.messages.stream(
    model="claude-3-5-sonnet-20241022",
    max_tokens=1024,
    system="你是一个诗人",
    messages=[{"role": "user", "content": "写一首诗"}]
) as stream:
    for text in stream.text_stream:
        print(text, end="")
```

**差异说明**：
- OpenAI 用 `stream=True` + 迭代 chunk
- Claude 用 `client.messages.stream()` 上下文管理器 + `text_stream` 迭代器
- Claude 的 stream API 更简洁，直接拿到文本流，不用层层解析 chunk

### J.2.5 Tool Calling 迁移

**Before（OpenAI SDK）：**

```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "获取指定城市的天气",
            "parameters": {
                "type": "object",
                "properties": {
                    "city": {"type": "string", "description": "城市名"}
                },
                "required": ["city"]
            }
        }
    }
]

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "北京天气怎么样？"}],
    tools=tools
)

tool_call = response.choices[0].message.tool_calls[0]
print(tool_call.function.name)      # "get_weather"
print(tool_call.function.arguments)  # '{"city": "北京"}'
```

**After（Claude Code SDK）：**

```python
tools = [
    {
        "name": "get_weather",
        "description": "获取指定城市的天气",
        "input_schema": {
            "type": "object",
            "properties": {
                "city": {"type": "string", "description": "城市名"}
            },
            "required": ["city"]
        }
    }
]

response = client.messages.create(
    model="claude-3-5-sonnet-20241022",
    max_tokens=1024,
    messages=[{"role": "user", "content": "北京天气怎么样？"}],
    tools=tools
)

# Claude 的 tool_use 内容在 content 数组中
for block in response.content:
    if block.type == "tool_use":
        print(block.name)       # "get_weather"
        print(block.input)       # {"city": "北京"}
```

**关键差异：**

| 概念 | OpenAI SDK | Claude Code SDK |
|------|-----------|----------------|
| Tool 定义格式 | `type: "function"` 嵌套 | 扁平结构，直接定义 `name`/`description`/`input_schema` |
| Tool 调用结果 | `message.tool_calls[0].function` | `content` 数组中 `block.type == "tool_use"` |
| 参数 schema | `parameters` | `input_schema` |
| 多 Tool 并行调用 | 支持（1 个 message 多个 tool_call） | 支持（1 个 response 多个 `tool_use` block） |

### J.2.6 完整迁移示例（可运行）

```python
import anthropic
import os
from dotenv import load_dotenv

load_dotenv()
client = anthropic.Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

# Tool 定义
tools = [
    {
        "name": "calculate",
        "description": "计算数学表达式",
        "input_schema": {
            "type": "object",
            "properties": {
                "expression": {"type": "string", "description": "数学表达式，如 '2+3*5'"}
            },
            "required": ["expression"]
        }
    }
]

# 多轮对话
messages = [
    {"role": "user", "content": "帮我算一下 (10 + 5) * 3"}
]

response = client.messages.create(
    model="claude-3-5-sonnet-20241022",
    max_tokens=1024,
    system="你是一个数学助手，善于使用工具进行计算。",
    messages=messages,
    tools=tools
)

# 处理回复
for block in response.content:
    if block.type == "text":
        print("Claude:", block.text)
    elif block.type == "tool_use":
        print(f"Claude 想调用工具: {block.name}")
        print(f"参数: {block.input}")
```

---

## J.3 路径二：从 OpenAI Node.js SDK 迁移

### J.3.1 安装依赖

```bash
npm uninstall openai
npm install @anthropic-ai/sdk
```

### J.3.2 基础聊天迁移

**Before（OpenAI SDK）：**

```typescript
import OpenAI from 'openai';

const client = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

const response = await client.chat.completions.create({
  model: 'gpt-4o',
  messages: [
    { role: 'system', content: '你是一个编程助手' },
    { role: 'user', content: '用 TypeScript 写一个快速排序' }
  ]
});

console.log(response.choices[0].message.content);
```

**After（Claude Code SDK）：**

```typescript
import Anthropic from '@anthropic-ai/sdk';

const client = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

const response = await client.messages.create({
  model: 'claude-3-5-sonnet-20241022',
  max_tokens: 1024,
  system: '你是一个编程助手',
  messages: [
    { role: 'user', content: '用 TypeScript 写一个快速排序' }
  ]
});

console.log(response.content[0].text);
```

### J.3.3 TypeScript 类型定义迁移

OpenAI SDK 的类型系统很完善，Claude Code SDK 也有完整类型支持。

**Before（OpenAI SDK 类型）：**

```typescript
import OpenAI from 'openai';

// 请求类型
const params: OpenAI.Chat.ChatCompletionCreateParams = {
  model: 'gpt-4o',
  messages: [{ role: 'user', content: 'Hello' }]
};

// 响应类型
const response: OpenAI.Chat.ChatCompletion = await client.chat.completions.create(params);
```

**After（Claude Code SDK 类型）：**

```typescript
import Anthropic from '@anthropic-ai/sdk';

// 请求类型
const params: Anthropic.Messages.MessageCreateParams = {
  model: 'claude-3-5-sonnet-20241022',
  max_tokens: 1024,
  messages: [{ role: 'user', content: 'Hello' }]
};

// 响应类型
const response: Anthropic.Messages.Message = await client.messages.create(params);
```

### J.3.4 Tool Definition 迁移（Node.js）

**Before（OpenAI SDK）：**

```typescript
const tools: OpenAI.Chat.Completions.ChatCompletionTool[] = [
  {
    type: 'function',
    function: {
      name: 'get_weather',
      description: '获取城市天气',
      parameters: {
        type: 'object',
        properties: {
          city: { type: 'string' }
        },
        required: ['city']
      }
    }
  }
];
```

**After（Claude Code SDK）：**

```typescript
import { Tool } from '@anthropic-ai/sdk';

const tools: Tool[] = [
  {
    name: 'get_weather',
    description: '获取城市天气',
    input_schema: {
      type: 'object',
      properties: {
        city: { type: 'string', description: '城市名' }
      },
      required: ['city']
    }
  }
];
```

### J.3.5 Streaming 迁移（Node.js）

**Before（OpenAI SDK）：**

```typescript
const stream = await client.chat.completions.create({
  model: 'gpt-4o',
  messages: [{ role: 'user', content: '写一首诗' }],
  stream: true
});

for await (const chunk of stream) {
  const content = chunk.choices[0]?.delta?.content;
  if (content) process.stdout.write(content);
}
```

**After（Claude Code SDK）：**

```typescript
const stream = client.messages.stream({
  model: 'claude-3-5-sonnet-20241022',
  max_tokens: 1024,
  messages: [{ role: 'user', content: '写一首诗' }]
});

for await (const text of stream.textStream) {
  process.stdout.write(text);
}
```

---

## J.4 路径三：从 LangChain 迁移到 Claude Code SDK

LangChain 是一个编排框架，它本身支持多个底层模型。你有**两种迁移策略**：

| 策略 | 做法 | 适用场景 |
|------|------|----------|
| **策略 A：保留 LangChain，换底层模型** | 把 `ChatOpenAI` 换成 `ChatAnthropic` | 想保留 LangChain 的 Tool/LangGraph 生态 |
| **策略 B：完全迁移到 Claude Code SDK** | 移除 LangChain，直接用 `@anthropic-ai/sdk` | 追求更轻量、更可控、更低抽象层 |

### J.4.1 策略 A：保留 LangChain，换底层模型

这是**最省事的迁移**，只需要换一个 import。

```typescript
// Before: LangChain + OpenAI
import { ChatOpenAI } from "@langchain/openai";

const model = new ChatOpenAI({
  modelName: "gpt-4o",
  openAIApiKey: process.env.OPENAI_API_KEY
});

const response = await model.invoke("用 Python 写快速排序");
console.log(response.content);
```

```typescript
// After: LangChain + Claude
import { ChatAnthropic } from "@langchain/anthropic";

const model = new ChatAnthropic({
  modelName: "claude-3-5-sonnet-20241022",
  anthropicApiKey: process.env.ANTHROPIC_API_KEY
});

const response = await model.invoke("用 Python 写快速排序");
console.log(response.content);
```

**安装依赖：**

```bash
npm install @langchain/anthropic
```

**优势**：
- LangChain 的 Tool、Memory、Chain 全部保留
- LangGraph 多 Agent 编排不变
- 只需改初始化代码，改动量极小

### J.4.2 策略 B：完全迁移到 Claude Code SDK

如果你想彻底摆脱 LangChain 的抽象层（更轻量、更可控），直接换到原生 Claude Code SDK。

**Before（LangChain Agent）：**

```typescript
import { ChatOpenAI } from "@langchain/openai";
import { createOpenAIFunctionsAgent } from "langchain/agents";

const model = new ChatOpenAI({ modelName: "gpt-4o" });
const agent = await createOpenAIFunctionsAgent({ llm: model, tools });
```

**After（Claude Code SDK 原生 Tool Use）：**

```typescript
import Anthropic from '@anthropic-ai/sdk';

const client = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

const tools = [
  {
    name: "get_weather",
    description: "获取城市天气",
    input_schema: { /* ... */ }
  }
];

const response = await client.messages.create({
  model: "claude-3-5-sonnet-20241022",
  max_tokens: 1024,
  messages: [{ role: "user", content: "北京天气怎么样？" }],
  tools
});
```

**什么时候选策略 B？**
- 你发现 LangChain 的抽象反而让调试变困难
- 你需要更精细地控制 Token 使用和成本
- 你只需要基础 Chat + Tool Use，不需要 LangChain 的高级编排

---

## J.5 常见问题与坑点

### Q1: Claude 的 `system` 参数和 OpenAI 的 `system` 消息有什么区别？

**OpenAI**：system 是 `messages` 数组里的一个元素：
```python
messages = [
    {"role": "system", "content": "你是一个助手"},
    {"role": "user", "content": "你好"}
]
```

**Claude**：system 是**独立参数**，不在 `messages` 里：
```python
response = client.messages.create(
    system="你是一个助手",  # 独立参数！
    messages=[{"role": "user", "content": "你好"}]  # 不含 system
)
```

**迁移陷阱**：如果你直接把 OpenAI 的 `messages` 数组传给 Claude，`{"role": "system"}` 那条消息会被 Claude 当成 `user` 消息处理，导致行为异常。

### Q2: Claude 支持 `temperature` 参数吗？

支持，但范围不同：

| 参数 | OpenAI | Claude Code SDK |
|------|--------|------------------|
| `temperature` | 0.0 ~ 2.0 | 0.0 ~ 1.0（建议 0.0~0.7） |
| `top_p` | 支持 | 支持 |
| `max_tokens` | 支持 | `max_tokens`（注意：Claude 3.5 需要**必填** `max_tokens`） |

### Q3: 模型名称怎么对应？

```python
# OpenAI 模型名
"gpt-4o"
"gpt-4-turbo"
"gpt-3.5-turbo"

# Claude 模型名（Anthropic 官方）
"claude-3-5-sonnet-20241022"   # 最新 Sonnet（推荐）
"claude-3-5-haiku-20241022"    # 最新 Haiku（快速便宜）
"claude-opus-4-20250514"       # 最新 Opus（最强最大）
```

> **注意**：Claude 的模型名带日期后缀（如 `20241022`），这是 Anthropic 的版本管理机制。你必须指定具体版本，不能用 `latest` 别名。

### Q4: 迁移后输出质量下降了？

可能原因：
1. **System Prompt 没正确迁移**（见 Q1）
2. **Max Tokens 设置太小**：Claude 默认需要显式设置 `max_tokens`（建议 4096 或更高）
3. **Prompt 格式差异**：Claude 对 XML 格式 Prompt 支持更好，可以尝试把 System Prompt 改成 XML 结构：

```python
system = """
<role>你是一个 Python 编程专家</role>
<rules>
- 代码必须包含类型注解
- 必须使用 asyncio
</rules>
"""
```

### Q5: Tool Calling 时 Claude 总是拒答？

Claude 的拒答率比 GPT-4o 低，但如果你遇到拒答：
1. 检查 Tool 的 `description` 是否清晰（Claude 依赖 description 决定何时调用 Tool）
2. 在 System Prompt 里明确告诉 Claude 可以/应该使用 Tool：

```python
system = """你是一个助手，可以调用以下工具：
- get_weather: 获取天气
- calculate: 计算数学表达式

当用户问天气或需要计算时，必须先调用对应工具，不要直接回答。"""
```

---

## J.6 迁移检查清单

完成了以下所有项，你的迁移就算成功了：

- [ ] 依赖已切换（`openai` → `anthropic` 或 `@anthropic-ai/sdk`）
- [ ] 客户端初始化代码已更新
- [ ] 所有 `chat.completions.create` 已替换为 `messages.create`
- [ ] System Prompt 从 `messages` 数组移到独立 `system` 参数
- [ ] 响应解析代码已更新（`choices[0].message.content` → `content[0].text`）
- [ ] Tool 定义已从 `function` 嵌套格式迁移到扁平 `input_schema` 格式
- [ ] Stream 代码已迁移（`stream=True` → `messages.stream()`）
- [ ] 单元测试已更新（Mock 对象需要同步修改）
- [ ] 成本监控已切换（OpenAI 的 `usage.prompt_tokens` → Claude 的 `usage.input_tokens`）
- [ ] 错误重试逻辑已更新（Anthropic 的错误码和 OpenAI 不同）

---

## J.7 总结

| 迁移路径 | 难度 | 推荐场景 |
|---------|------|----------|
| OpenAI Python SDK → Claude Python SDK | ⭐⭐ | 纯 Python 项目 |
| OpenAI Node.js SDK → Claude Node.js SDK | ⭐⭐ | TypeScript/Node.js 项目 |
| LangChain (OpenAI) → LangChain (Anthropic) | ⭐ | 想保留 LangChain 生态 |
| LangChain (OpenAI) → Claude SDK 原生 | ⭐⭐⭐ | 追求轻量可控 |

**核心要点记住三条：**
1. System Prompt 是独立参数，不是 `messages` 的一部分
2. Tool 定义用 `input_schema`，不是 `parameters`
3. 响应内容在 `content[0].text`，不是 `choices[0].message.content`

搞定这三条，迁移就成功了 80%。剩下的是细节打磨。

> 下一章（附录K）：Claude Code SDK 安全合规指南 —— 企业落地的必经之路。
