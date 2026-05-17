# 第4章：核心概念与架构

> 本章介绍 Claude Code SDK 的核心架构概念、主要接口类型和异步编程模型。理解这些基础概念是高效使用 SDK 的前提。

## 4.1 SDK 架构概览

Claude Code SDK 是一个用于构建 AI 编程助手的开发工具包。它让开发者能够在自己的应用程序中集成 Claude 的编程能力，实现自动化代码编辑、命令执行和工具调用等功能。

### 4.1.1 整体架构

SDK 的架构可以分为以下几个层次：

```
┌─────────────────────────────────────────────────────┐
│                   应用层                            │
│         (你编写的应用程序代码)                       │
├─────────────────────────────────────────────────────┤
│                   SDK 层                           │
│    Client / Session / Message / Tools                │
├─────────────────────────────────────────────────────┤
│                   传输层                           │
│        HTTP / WebSocket / Stdio                    │
├─────────────────────────────────────────────────────┤
│                   API 层                           │
│          Anthropic API (Claude AI)                   │
└─────────────────────────────────────────────────────┘
```

### 4.1.2 核心组件

SDK 的核心组件包括：

1. **Client** - 客户端连接管理
2. **Session** - 对话会话管理
3. **Message** - 消息构建与解析
4. **Tool** - 工具定义与调用
5. **MCP** - 外部工具集成

## 4.2 主要接口和类型

### 4.2.1 Client 类型

Client 是 SDK 的入口点，用于建立与 Claude Code 的连接。

```typescript
import { Client } from '@anthropic-ai/claude-code-sdk';

// 创建客户端实例
const client = new Client({
  // API 配置
  apiKey: process.env.ANTHROPIC_API_KEY,
  model: 'claude-sonnet-4-20250514',
  
  // 可选：自定义基础 URL（用于本地或代理环境）
  baseUrl: 'https://api.anthropic.com',
  
  // 可选：最大请求超时（毫秒）
  maxRetries: 3,
});

// 启动新会话
const session = await client.sessions.create({
  systemPrompt: '你是一个专业的编程助手。',
});
```

### 4.2.2 Message 结构

Message 是 SDK 中的核心数据结构，用于表示对话消息。

```typescript
// 消息类型定义
interface Message {
  // 消息角色：user（用户）、assistant（助手）、system（系统）
  role: 'user' | 'assistant' | 'system';
  
  // 消息内容（Content Block 数组）
  content: ContentBlock[];
  
  // 可选：用于追踪的工具调用 ID
  toolCallId?: string;
  
  // 可选：时间戳
  timestamp?: number;
}

// Content Block 类型
type ContentBlock = 
  | TextContent    // 文本内容
  | ToolUseContent // 工具调用请求
  | ToolResultContent; // 工具执行结果
```

### 4.2.3 Content Block 详解

```typescript
// 文本内容
interface TextContent {
  type: 'text';
  text: string;
}

// 工具调用请求（模型要求调用工具）
interface ToolUseContent {
  type: 'tool_use';
  id: string;           // 唯一标识符
  name: string;        // 工具名称
  input: object;       // 工具输入参数
}

// 工具执行结果
interface ToolResultContent {
  type: 'tool_result';
  toolUseId: string;   // 对应的工具调用 ID
  content: string;   // 执行结果
  isError?: boolean;  // 是否是错误
}
```

### 4.2.4 Tool 定义

Tool 是让 Claude Code 能够执行自定义操作的机制。

```typescript
// Tool 定义
interface Tool {
  // 工具名称（必须唯一）
  name: string;
  
  // 工具描述（帮助模型理解何时使用）
  description: string;
  
  // JSON Schema 格式的参数定义
  inputSchema: {
    type: 'object';
    properties: {
      [key: string]: {
        type: string;
        description: string;
      };
    };
    required: string[];
  };
}

// 示例：定义一个文件读取工具
const readFileTool: Tool = {
  name: 'read_file',
  description: '读取指定路径的文件内容',
  inputSchema: {
    type: 'object',
    properties: {
      path: {
        type: 'string',
        description: '要读取的文件路径',
      },
      lines: {
        type: 'number',
        description: '要读取的行数（可选）',
      },
    },
    required: ['path'],
  },
};

// 注册工具到会话
session.registerTools([readFileTool]);

// 定义工具处理函数
session.handleTool('read_file', async (input) => {
  const fs = await import('fs/promises');
  const content = await fs.readFile(input.path, 'utf-8');
  return { content };
});
```

## 4.3 异步编程模型

Claude Code SDK 完全基于异步编程模型，充分利用 JavaScript/TypeScript 的 async/await 特性。

### 4.3.1 基本异步流程

```typescript
import { Client } from '@anthropic-ai/claude-code-sdk';

async function main() {
  const client = new Client({
    apiKey: process.env.ANTHROPIC_API_KEY!,
  });

  // 创建会话
  const session = await client.sessions.create({
    systemPrompt: '你是一个代码审查助手。',
  });

  // 发送消息并等待响应
  const response = await session.send({
    role: 'user',
    content: [{ type: 'text', text: '帮我审查 ~/project/src/index.ts' }],
  });

  console.log('Claude 的回复:', response.content);
}

main().catch(console.error);
```

### 4.3.2 流式响应

对于长回复，可以使用流式处理减少延迟：

```typescript
import { Client } from '@anthropic-ai/claude-code-sdk';

async function streamChat() {
  const client = new Client({
    apiKey: process.env.ANTHROPIC_API_KEY!,
  });

  const session = await client.sessions.create();

  // 使用流式响应
  const stream = await session.send({
    role: 'user',
    content: [
      { 
        type: 'text', 
        text: '写一个排序算法的实现' 
      },
    ],
    stream: true,  // 启用流式
  });

  // 处理流式数据块
  for await (const chunk of stream) {
    if (chunk.type === 'content') {
      process.stdout.write(chunk.text);
    }
  }
}
```

### 4.3.3 并发工具调用

SDK 支持并发处理多个工具调用，提高效率：

```typescript
import { Client } from '@anthropic-ai/claude-code-sdk';

async function parallelToolCalls() {
  const client = new Client({
    apiKey: process.env.ANTHROPIC_API_KEY!,
  });

  const session = await client.sessions.create();

  // 注册多个工具
  session.registerTools([
    {
      name: 'read_file',
      description: '读取文件',
      inputSchema: { /* ... */ },
    },
    {
      name: 'run_command',
      description: '运行命令',
      inputSchema: { /* ... */ },
    },
  ]);

  // 处理工具调用（可以并发执行）
  session.handleTools(async (toolCalls) => {
    // 使用 Promise.all 并发执行
    const results = await Promise.all(
      toolCalls.map(async (call) => {
        switch (call.name) {
          case 'read_file':
            return { result: await readFile(call.input.path) };
          case 'run_command':
            return { result: await runCommand(call.input.cmd) };
        }
      })
    );
    return results;
  });
}
```

### 4.3.4 错误处理与重试

```typescript
import { Client, RetryConfig } from '@anthropic-ai/claude-code-sdk';

const retryConfig: RetryConfig = {
  maxRetries: 3,
  initialDelayMs: 1000,
  maxDelayMs: 10000,
  backoffMultiplier: 2,
};

const client = new Client({
  apiKey: process.env.ANTHROPIC_API_KEY!,
  retryConfig,
});

// 异常处理示例
try {
  const session = await client.sessions.create();
  const response = await session.send({ /* ... */ });
} catch (error) {
  if (error instanceof RateLimitError) {
    console.log('速率限制，等待重试...');
    await sleep(error.retryAfter * 1000);
  } else if (error instanceof AuthenticationError) {
    console.log('认证失败，请检查 API Key');
  } else if (error instanceof APIError) {
    console.log(`API 错误: ${error.message}`);
  } else {
    throw error;
  }
}
```

## 4.4 小结

本章介绍了 Claude Code SDK 的核心架构：

1. **SDK 架构** - 分层架构，包括应用层、SDK层、传输层和API层
2. **主要接口和类型** - Client、Session、Message、Content Block、Tool
3. **异步编程模型** - 异步流程、流式响应、并发处理、错误重试

掌握这些概念后，你就可以开始构建自己的 AI 编程助手了。下一章我们将深入了解 Message 和 Content 类型的详细用法。

---

**本章代码示例关键词**：`Client`、`Session`、`Message`、`Content Block`、`Tool`、`async/await`、`stream`、`retry`