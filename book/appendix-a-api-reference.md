# 附录A：API 参考手册

> 本附录汇总 Claude Code SDK 的核心 API、类型定义和常用模式，供开发者快速查阅。

---

## A.1 核心类型速查表

### A.1.1 Message 类型

```typescript
interface Message {
  role: 'user' | 'assistant' | 'system';
  content: ContentBlock[];
}
```

| 字段 | 类型 | 必填 | 说明 |
|-----|------|------|------|
| `role` | `'user'` \| `'assistant'` \| `'system'` | ✅ | 消息发送者角色 |
| `content` | `ContentBlock[]` | ✅ | 消息内容块数组 |

**使用示例：**

```typescript
// 用户消息
const userMessage: Message = {
  role: 'user',
  content: [
    { type: 'text', text: '你好，Claude！' }
  ]
};

// 助手消息
const assistantMessage: Message = {
  role: 'assistant',
  content: [
    { type: 'text', text: '你好！有什么我可以帮助你的吗？' }
  ]
};
```

---

### A.1.2 ContentBlock 类型

ContentBlock 是联合类型，包含四种具体类型：

```typescript
type ContentBlock = 
  | TextBlock 
  | ImageBlock 
  | ToolUseBlock 
  | ToolResultBlock;
```

#### TextBlock（文本内容）

```typescript
interface TextBlock {
  type: 'text';
  text: string;
}
```

| 字段 | 类型 | 必填 | 说明 |
|-----|------|------|------|
| `type` | `'text'` | ✅ | 固定值 |
| `text` | `string` | ✅ | 文本内容 |

#### ImageBlock（图片内容）

```typescript
interface ImageBlock {
  type: 'image';
  source: {
    type: 'base64' | 'url';
    media_type: 'image/jpeg' | 'image/png' | 'image/gif' | 'image/webp';
    data?: string;   // base64 编码数据
    url?: string;    // 图片 URL
  };
}
```

| 字段 | 类型 | 必填 | 说明 |
|-----|------|------|------|
| `type` | `'image'` | ✅ | 固定值 |
| `source.type` | `'base64'` \| `'url'` | ✅ | 图片来源类型 |
| `source.media_type` | `string` | ✅ | 图片 MIME 类型 |
| `source.data` | `string` | 条件 | Base64 编码的图片数据（type=base64 时必填） |
| `source.url` | `string` | 条件 | 图片 URL（type=url 时必填） |

**图片大小限制：** 单张图片最大约 5MB。

#### ToolUseBlock（工具调用请求）

```typescript
interface ToolUseBlock {
  type: 'tool_use';
  id: string;       // 唯一标识符
  name: string;     // 工具名称
  input: object;    // 工具输入参数
}
```

| 字段 | 类型 | 必填 | 说明 |
|-----|------|------|------|
| `type` | `'tool_use'` | ✅ | 固定值 |
| `id` | `string` | ✅ | 工具调用的唯一标识符（由 Claude 生成） |
| `name` | `string` | ✅ | 工具名称 |
| `input` | `object` | ✅ | 工具输入参数（JSON 对象） |

#### ToolResultBlock（工具执行结果）

```typescript
interface ToolResultBlock {
  type: 'tool_result';
  tool_use_id: string;  // 对应的 tool_use.id
  content: string;      // 执行结果
  is_error?: boolean;   // 是否为错误
}
```

| 字段 | 类型 | 必填 | 说明 |
|-----|------|------|------|
| `type` | `'tool_result'` | ✅ | 固定值 |
| `tool_use_id` | `string` | ✅ | 对应的 tool_use 调用的 ID |
| `content` | `string` | ✅ | 工具执行结果（或错误信息） |
| `is_error` | `boolean` | ❌ | 是否为错误结果，默认 `false` |

---

### A.1.3 Tool 定义

```typescript
interface Tool {
  name: string;           // 工具名称
  description: string;    // 工具描述
  input_schema: {         // 参数 Schema
    type: 'object';
    properties: Record<string, JsonSchemaProperty>;
    required?: string[];
  };
}

interface JsonSchemaProperty {
  type: 'string' | 'number' | 'integer' | 'boolean' | 'array' | 'object';
  description?: string;
  enum?: string[] | number[];
  items?: JsonSchemaProperty;        // 数组元素类型
  properties?: Record<string, JsonSchemaProperty>;  // 嵌套对象
  required?: string[];
  default?: any;
  minimum?: number;
  maximum?: number;
  minLength?: number;
  maxLength?: number;
  maxItems?: number;
}
```

| 字段 | 类型 | 必填 | 说明 |
|-----|------|------|------|
| `name` | `string` | ✅ | 工具名称（字母、数字、下划线、连字符） |
| `description` | `string` | ✅ | 工具描述（帮助 Claude 决定何时调用） |
| `input_schema` | `object` | ✅ | JSON Schema 格式的参数定义 |

---

## A.2 SDK 客户端 API

### A.2.1 ClaudeCode 类

```typescript
import { ClaudeCode } from '@anthropic-ai/claude-code-sdk';

const claude = new ClaudeCode({
  apiKey?: string;        // API 密钥（可从环境变量读取）
  model?: string;         // 默认模型
  baseUrl?: string;       // API 基础 URL
  maxRetries?: number;    // 最大重试次数
  timeout?: number;       // 请求超时（毫秒）
});
```

**配置选项：**

| 参数 | 类型 | 默认值 | 说明 |
|-----|------|--------|------|
| `apiKey` | `string` | `process.env.ANTHROPIC_API_KEY` | API 密钥 |
| `model` | `string` | `'claude-sonnet-4-20250514'` | 默认模型 |
| `baseUrl` | `string` | `'https://api.anthropic.com'` | API 基础 URL |
| `maxRetries` | `number` | `3` | 最大重试次数 |
| `timeout` | `number` | `60000` | 请求超时（毫秒） |

---

### A.2.2 messages.create() 方法

创建消息对话的核心方法：

```typescript
const response = await claude.messages.create({
  model: string;              // 模型名称
  maxTokens: number;          // 最大输出 Token 数
  messages: Message[];        // 消息历史
  tools?: Tool[];             // 可用工具列表
  system?: string;            // 系统提示词
  temperature?: number;       // 温度参数（0-1）
  stopSequences?: string[];   // 停止序列
  stream?: boolean;           // 是否流式输出
});
```

**参数详解：**

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|-----|------|------|--------|------|
| `model` | `string` | ✅ | - | 模型名称，见 A.3 节 |
| `maxTokens` | `number` | ✅ | - | 最大输出 Token 数（1-4096） |
| `messages` | `Message[]` | ✅ | - | 对话消息数组 |
| `tools` | `Tool[]` | ❌ | `[]` | 可用工具列表 |
| `system` | `string` | ❌ | `''` | 系统提示词 |
| `temperature` | `number` | ❌ | `1.0` | 随机性控制（0-1） |
| `stopSequences` | `string[]` | ❌ | `[]` | 遇到这些序列时停止生成 |
| `stream` | `boolean` | ❌ | `false` | 是否启用流式输出 |

**返回值：**

```typescript
interface MessageResponse {
  id: string;               // 响应 ID
  type: 'message';
  role: 'assistant';
  content: ContentBlock[];  // 生成的内容
  model: string;            // 使用的模型
  stopReason: 'end_turn' | 'max_tokens' | 'stop_sequence' | 'tool_use';
  usage: {
    inputTokens: number;
    outputTokens: number;
  };
}
```

---

### A.2.3 流式响应处理

启用 `stream: true` 后，返回异步迭代器：

```typescript
const stream = await claude.messages.create({
  model: 'claude-sonnet-4-20250514',
  maxTokens: 1024,
  messages: [{ role: 'user', content: '讲个笑话' }],
  stream: true,
});

for await (const chunk of stream) {
  if (chunk.type === 'content_block_delta') {
    process.stdout.write(chunk.delta.text);
  }
}
```

**流式事件类型：**

| 事件类型 | 说明 |
|---------|------|
| `message_start` | 消息开始 |
| `content_block_start` | 内容块开始 |
| `content_block_delta` | 内容块增量 |
| `content_block_stop` | 内容块结束 |
| `message_delta` | 消息增量（如 stop_reason） |
| `message_stop` | 消息结束 |

---

## A.3 支持的模型列表

### A.3.1 推荐模型

| 模型名称 | 特点 | 适用场景 | 相对成本 |
|---------|------|---------|---------|
| `claude-sonnet-4-20250514` | 平衡性能与速度 | 日常开发、通用任务 | ⭐⭐ |
| `claude-opus-4-20250514` | 最强性能 | 复杂推理、代码重构 | ⭐⭐⭐ |
| `claude-haiku-3-5-20241022` | 最快速度 | 简单任务、快速补全 | ⭐ |

### A.3.2 模型选择建议

```typescript
// 高质量代码生成
const response = await claude.messages.create({
  model: 'claude-opus-4-20250514',
  maxTokens: 4096,
  messages: [{ role: 'user', content: '重构这个模块...' }],
});

// 快速响应场景
const response = await claude.messages.create({
  model: 'claude-haiku-3-5-20241022',
  maxTokens: 1024,
  messages: [{ role: 'user', content: '补充这个函数的注释' }],
});
```

---

## A.4 错误类型与处理

### A.4.1 错误类型定义

```typescript
// 基础错误类
class ClaudeCodeError extends Error {
  code: string;
  statusCode?: number;
}

// 认证错误
class AuthenticationError extends ClaudeCodeError {
  code = 'authentication_error';
}

// 速率限制错误
class RateLimitError extends ClaudeCodeError {
  code = 'rate_limit_error';
  retryAfter: number;  // 重试等待时间（秒）
}

// API 错误
class APIError extends ClaudeCodeError {
  code = 'api_error';
}

// 无效请求错误
class InvalidRequestError extends ClaudeCodeError {
  code = 'invalid_request_error';
}
```

### A.4.2 错误处理示例

```typescript
import { 
  ClaudeCode, 
  AuthenticationError, 
  RateLimitError, 
  APIError 
} from '@anthropic-ai/claude-code-sdk';

async function safeChat(prompt: string) {
  try {
    const response = await claude.messages.create({
      model: 'claude-sonnet-4-20250514',
      maxTokens: 1024,
      messages: [{ role: 'user', content: prompt }],
    });
    return response;
  } catch (error) {
    if (error instanceof AuthenticationError) {
      console.error('认证失败：请检查 API Key');
      throw new Error('请设置有效的 ANTHROPIC_API_KEY');
    } else if (error instanceof RateLimitError) {
      console.error(`速率限制：请等待 ${error.retryAfter} 秒后重试`);
      await sleep(error.retryAfter * 1000);
      return safeChat(prompt);  // 重试
    } else if (error instanceof APIError) {
      console.error(`API 错误：${error.message}`);
      throw error;
    } else {
      throw error;
    }
  }
}
```

---

## A.5 工具定义模式速查

### A.5.1 基础类型参数

```typescript
// 字符串参数
properties: {
  name: { type: 'string', description: '用户名称' }
}

// 数字参数
properties: {
  age: { type: 'integer', description: '用户年龄', minimum: 0, maximum: 150 }
}

// 布尔参数
properties: {
  enabled: { type: 'boolean', description: '是否启用', default: true }
}
```

### A.5.2 枚举参数

```typescript
properties: {
  unit: {
    type: 'string',
    enum: ['celsius', 'fahrenheit'],
    description: '温度单位'
  }
}
```

### A.5.3 数组参数

```typescript
properties: {
  tags: {
    type: 'array',
    items: { type: 'string' },
    description: '标签列表',
    maxItems: 10
  }
}
```

### A.5.4 嵌套对象参数

```typescript
properties: {
  options: {
    type: 'object',
    description: '搜索选项',
    properties: {
      caseSensitive: { type: 'boolean', description: '区分大小写' },
      maxResults: { type: 'integer', description: '最大结果数' }
    }
  }
}
```

### A.5.5 可选参数 + 默认值

```typescript
properties: {
  format: {
    type: 'string',
    enum: ['json', 'xml', 'yaml'],
    default: 'json',
    description: '输出格式'
  }
},
required: ['query']  // format 不在 required 中，所以是可选的
```

---

## A.6 常用代码模板

### A.6.1 基础对话模板

```typescript
import { ClaudeCode } from '@anthropic-ai/claude-code-sdk';

const claude = new ClaudeCode();

async function chat(userInput: string) {
  const response = await claude.messages.create({
    model: 'claude-sonnet-4-20250514',
    maxTokens: 1024,
    messages: [{ role: 'user', content: userInput }],
  });

  return response.content
    .filter(b => b.type === 'text')
    .map(b => b.text)
    .join('');
}
```

### A.6.2 多轮对话模板

```typescript
const messages: Message[] = [];

async function multiTurnChat(userInput: string) {
  // 添加用户消息
  messages.push({ role: 'user', content: userInput });

  // 获取 Claude 响应
  const response = await claude.messages.create({
    model: 'claude-sonnet-4-20250514',
    maxTokens: 2048,
    messages: messages,
  });

  // 保存助手消息
  messages.push({ role: 'assistant', content: response.content });

  return response.content
    .filter(b => b.type === 'text')
    .map(b => b.text)
    .join('');
}
```

### A.6.3 工具调用模板

```typescript
const tools: Tool[] = [
  {
    name: 'get_weather',
    description: '获取天气信息',
    input_schema: {
      type: 'object',
      properties: {
        city: { type: 'string', description: '城市名称' }
      },
      required: ['city']
    }
  }
];

const toolHandlers: Record<string, (input: any) => Promise<string>> = {
  get_weather: async (input) => {
    // 实际实现
    return `${input.city}：晴天，25°C`;
  }
};

async function toolChat(prompt: string) {
  const messages: any[] = [{ role: 'user', content: prompt }];

  let response = await claude.messages.create({
    model: 'claude-sonnet-4-20250514',
    maxTokens: 2048,
    tools,
    messages,
  });

  // 处理工具调用
  const toolUses = response.content.filter(b => b.type === 'tool_use');
  
  if (toolUses.length > 0) {
    messages.push({ role: 'assistant', content: response.content });

    // 并发执行工具
    const toolResults = await Promise.all(
      toolUses.map(async (toolUse: any) => {
        const handler = toolHandlers[toolUse.name];
        const result = handler 
          ? await handler(toolUse.input)
          : `未知工具：${toolUse.name}`;

        return {
          type: 'tool_result' as const,
          tool_use_id: toolUse.id,
          content: result,
        };
      })
    );

    messages.push({ role: 'user', content: toolResults });

    // 获取最终响应
    response = await claude.messages.create({
      model: 'claude-sonnet-4-20250514',
      maxTokens: 2048,
      tools,
      messages,
    });
  }

  return response.content
    .filter(b => b.type === 'text')
    .map(b => b.text)
    .join('');
}
```

### A.6.4 流式输出模板

```typescript
async function streamChat(prompt: string) {
  const stream = await claude.messages.create({
    model: 'claude-sonnet-4-20250514',
    maxTokens: 2048,
    messages: [{ role: 'user', content: prompt }],
    stream: true,
  });

  let fullText = '';

  for await (const chunk of stream) {
    if (chunk.type === 'content_block_delta') {
      const text = chunk.delta?.text || '';
      process.stdout.write(text);
      fullText += text;
    }
  }

  return fullText;
}
```

---

## A.7 CLI 命令速查

### A.7.1 基础命令

```bash
# 启动交互式会话
claude

# 单次提问
claude -p "你的问题"

# 从标准输入读取
echo "解释这段代码" | claude -p

# 指定模型
claude --model claude-opus-4-20250514

# 设置最大 Token
claude --max-tokens 4096

# JSON 输出格式
claude --output-format json
```

### A.7.2 权限模式

```bash
# 规划模式（只规划不执行）
claude /plan

# 接受模式（执行前需要确认）
claude /accept

# 自动模式（自动执行，谨慎使用）
claude /yolo
```

### A.7.3 MCP 相关

```bash
# 添加 MCP 服务器
claude mcp add <server-name> --command <command>

# 列出 MCP 服务器
claude mcp list

# 删除 MCP 服务器
claude mcp remove <server-name>
```

---

## A.8 环境变量

| 变量名 | 说明 | 默认值 |
|-------|------|--------|
| `ANTHROPIC_API_KEY` | API 密钥 | - |
| `CLAUDE_MODEL` | 默认模型 | `claude-sonnet-4-20250514` |
| `CLAUDE_MAX_TOKENS` | 默认最大 Token | `4096` |
| `CLAUDE_TEMPERATURE` | 默认温度 | `1.0` |
| `CLAUDE_BASE_URL` | API 基础 URL | `https://api.anthropic.com` |

---

## A.9 版本兼容性

### A.9.1 Node.js 版本

| SDK 版本 | 最低 Node.js 版本 | 推荐 Node.js 版本 |
|---------|------------------|------------------|
| 1.x | Node.js 18 | Node.js 20+ |
| 2.x | Node.js 20 | Node.js 22+ |

### A.9.2 TypeScript 版本

- 最低：TypeScript 4.9
- 推荐：TypeScript 5.0+

---

## A.10 常见问题速查

### Q1: 如何处理 `tool_use` 没有被调用？

**A:** Claude 会根据问题判断是否需要调用工具。如果工具描述不清晰，或问题不需要外部数据，Claude 可能直接回答文本。确保 `description` 清晰描述工具用途。

### Q2: 图片发送后响应很慢？

**A:** 图片大小影响处理速度。建议：
- 压缩图片至 1MB 以内
- 使用 JPEG 格式（比 PNG 更小）
- 限制图片宽度至 1600px 以内

### Q3: 如何控制 Token 消耗？

**A:** 
- 设置合理的 `maxTokens`
- 使用 `claude-haiku` 处理简单任务
- 清理不必要的对话历史
- 避免重复发送相同上下文

### Q4: 多个工具调用如何保证顺序？

**A:** Claude 可能同时调用多个工具，它们之间通常没有顺序依赖。如果有依赖，建议：
- 拆分成多次调用
- 在工具描述中说明依赖关系

---

## A.11 参考资料

- [Claude Code 官方文档](https://docs.anthropic.com/claude-code)
- [Anthropic API 文档](https://docs.anthropic.com/claude/reference)
- [JSON Schema 规范](https://json-schema.org/)
- [Model Context Protocol](https://modelcontextprotocol.io/)

---

**本附录持续更新中。如有遗漏或错误，欢迎反馈。**

---

_附录A 完，下一附录：常见问题解答 ➡️_
