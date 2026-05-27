# 第16章：插件开发指南

> "SDK 本身只是舞台，插件才是台上跳舞的演员。学会写插件，你就能让 Claude Code SDK 做任何事。" —— 老三

前面 15 章我们学会了如何用 Claude Code SDK 调用 Claude、定义 Tool、接入 MCP、做多 Agent 协作。但如果你遇到了 SDK 没覆盖到的场景——比如想接入公司内部系统、想自定义 Tool 的执行逻辑、想给 Claude 加一个专属知识库——这时候就需要**写插件**了。

这一章我们讲三件事：
1. **Claude Code SDK 的扩展点在哪**（搞清楚能从哪里下手）
2. **如何开发一个自定义 Tool 插件**（最实用的扩展方式）
3. **如何开发一个 MCP 插件**（让 Claude 能调用任何外部服务）

---

## 16.1 Claude Code SDK 的扩展架构

### 16.1.1 SDK 的核心扩展点

Claude Code SDK 的设计哲学是**可组合、可扩展**。它提供了三个主要的扩展点：

```
┌─────────────────────────────────────────────────┐
│             你的应用程序                         │
│  ┌─────────────────────────────────────────┐   │
│  │         Claude Code SDK                 │   │
│  │  ┌──────────┐  ┌──────────────────┐   │   │
│  │  │  Tool    │  │   MCP Client     │   │   │
│  │  │  Registry│  │   (MCP 协议)    │   │   │
│  │  └────┬─────┘  └────────┬─────────┘   │   │
│  │       │                  │             │   │
│  │  ┌────▼─────┐  ┌────────▼─────────┐   │   │
│  │  │ 自定义   │  │   自定义 MCP    │   │   │
│  │  │ Tool 插件│  │   服务器插件    │   │   │
│  │  └──────────┘  └──────────────────┘   │   │
│  └─────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

| 扩展点 | 适用场景 | 难度 | 灵活性 |
|--------|----------|------|--------|
| **自定义 Tool** | 给 Claude 加新能力（查数据库、调用内部 API） | ⭐⭐ | 高 |
| **MCP 服务器** | 让 Claude 调用标准 MCP 服务（文件系统、浏览器） | ⭐⭐⭐ | 最高 |
| **SDK 中间件** | 拦截/修改 SDK 的请求和响应 | ⭐⭐⭐⭐ | 中 |

### 16.1.2 插件 vs Tool vs MCP：怎么选？

很多同学会困惑：我想给 Claude 加一个"查天气"的功能，应该用插件、Tool、还是 MCP？

**决策树：**

```
你想加的功能是...
│
├─ 只在你的应用里用？
│   └─ 是 → 写自定义 Tool（简单直接）
│
├─ 想让多个应用都能用？
│   └─ 是 → 写 MCP 服务器（标准协议，可复用）
│
└─ 想修改 SDK 的请求/响应流程？
    └─ 是 → 写 SDK 中间件（高级玩法）
```

**结论：**
- **个人项目/快速原型** → 自定义 Tool
- **团队协作/开源共享** → MCP 服务器
- **需要深度定制 SDK 行为** → SDK 中间件

---

## 16.2 实战一：开发一个自定义 Tool 插件

### 16.2.1 场景：给 Claude 加上"公司知识库查询"能力

假设你在一家公司工作，公司有一个内部知识库（存着产品文档、API 规范、历史决策）。你希望 Claude 能直接查询这个知识库，而不是每次都手动粘贴文档。

**目标：** 写一个自定义 Tool，让 Claude 能调用 `company_kb_search(query: string)` 这个函数。

### 16.2.2 步骤一：定义 Tool Schema

Claude Code SDK 的 Tool 定义遵循 **Anthropic Tool Schema**（和 Claude API 一致）：

```typescript
// file: tools/company-kb-tool.ts

export const companyKbTool = {
  name: "company_kb_search",
  description: "查询公司知识库。输入关键词，返回相关文档片段。用于回答关于公司产品、API、历史决策的问题。",
  input_schema: {
    type: "object",
    properties: {
      query: {
        type: "string",
        description: "搜索关键词，例如：'API 认证流程'、'v2.0 发布计划'"
      }
    },
    required: ["query"]
  }
};
```

**关键点：**
- `name`：Tool 的唯一标识，Claude 会看到这个名字
- `description`：**非常重要！** Claude 根据这个描述决定是否调用 Tool。写得越清楚，Claude 越容易用对
- `input_schema`：用 JSON Schema 定义输入参数

### 16.2.3 步骤二：实现 Tool 执行逻辑

```typescript
// file: tools/company-kb-tool.ts（续）

// 模拟公司知识库（实际项目中可能是向量数据库 + RAG）
const COMPANY_KB = [
  {
    title: "API 认证流程",
    content: "我们使用 JWT 进行 API 认证。前端在登录后获取 token，后续请求在 Header 中携带 Authorization: Bearer <token>。"
  },
  {
    title: "v2.0 发布计划",
    content: "v2.0 将于 2026-Q3 发布，主要特性：1) 支持多模态输入 2) 新增 Batch API 3) 降价 30%"
  },
  {
    title: "数据库迁移规范",
    content: "所有数据库迁移必须使用 Prisma。禁止手写 SQL 迁移脚本。每次迁移前必须备份。"
  }
];

export async function executeCompanyKbSearch(query: string): Promise<string> {
  // 简单的关键词匹配（实际项目中用向量相似度搜索）
  const results = COMPANY_KB.filter(doc => 
    doc.title.includes(query) || doc.content.includes(query)
  );
  
  if (results.length === 0) {
    return "知识库中未找到相关内容。";
  }
  
  // 返回格式化的结果
  return results.map(doc => 
    `【${doc.title}】\n${doc.content}`
  ).join("\n\n");
}
```

### 16.2.4 步骤三：把 Tool 注册到 SDK

```typescript
// file: main.ts

import Anthropic from '@anthropic-ai/sdk';
import { companyKbTool, executeCompanyKbSearch } from './tools/company-kb-tool.js';

const client = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY!
});

async function chatWithKb(userMessage: string) {
  const response = await client.messages.create({
    model: "claude-sonnet-4-20250514",
    max_tokens: 1024,
    tools: [companyKbTool],  // ← 注册 Tool
    messages: [{ role: "user", content: userMessage }]
  });
  
  // 处理 Tool 调用
  for (const block of response.content) {
    if (block.type === "tool_use") {
      if (block.name === "company_kb_search") {
        const query = block.input.query as string;
        const result = await executeCompanyKbSearch(query);
        
        // 把 Tool 结果返回给 Claude
        const finalResponse = await client.messages.create({
          model: "claude-sonnet-4-20250514",
          max_tokens: 1024,
          tools: [companyKbTool],
          messages: [
            { role: "user", content: userMessage },
            { role: "assistant", content: response.content },
            { 
              role: "user", 
              content: [{ 
                type: "tool_result", 
                tool_use_id: block.id, 
                content: result 
              }] 
            }
          ]
        });
        
        console.log(finalResponse.content[0].text);
      }
    } else if (block.type === "text") {
      console.log(block.text);
    }
  }
}

// 测试
chatWithKb("我们公司的 API 认证流程是什么？");
```

**运行结果：**
```
我们公司使用 JWT 进行 API 认证。具体流程是：
1. 前端用户在登录时提供凭证
2. 登录成功后，服务器返回一个 JWT token
3. 后续所有 API 请求都需要在 HTTP Header 中携带这个 token：
   Authorization: Bearer <token>
```

### 16.2.5 进阶：给 Tool 加上错误处理和超时

生产环境中，Tool 执行可能会失败（网络超时、知识库宕机...）。你需要优雅地处理这些错误：

```typescript
// file: tools/company-kb-tool-v2.ts

export async function executeCompanyKbSearchV2(query: string): Promise<string> {
  try {
    // 设置超时
    const timeoutPromise = new Promise<never>((_, reject) => {
      setTimeout(() => reject(new Error("查询超时（3秒）")), 3000);
    });
    
    const searchPromise = executeCompanyKbSearch(query);
    
    const result = await Promise.race([searchPromise, timeoutPromise]);
    return result;
  } catch (error) {
    // 返回友好的错误信息（Claude 会根据这个信息决定是否重试）
    return `知识库查询失败：${error instanceof Error ? error.message : "未知错误"}。请稍后重试或联系管理员。`;
  }
}
```

**最佳实践：**
- Tool 执行时间超过 5 秒？加超时
- 网络请求失败？返回友好错误，别让 Claude 傻等
- 返回结果太长？截断到 1000 字以内（Claude 的上下文窗口有限）

---

## 16.3 实战二：开发一个 MCP 插件

### 16.3.1 为什么选 MCP？

MCP（Model Context Protocol）是 Anthropic 推出的**标准协议**，用于让 AI 模型调用外部工具和数据源。

**MCP 的优势：**
- **标准化**：一次写完，所有支持 MCP 的 AI 都能用（Claude、GPT、Gemini...）
- **可组合**：多个 MCP 服务器可以组合使用
- **生态丰富**：官方和社区已经有很多现成的 MCP 服务器（文件系统、数据库、浏览器...）

### 16.3.2 场景：写一个"天气查询" MCP 服务器

我们要写一个 MCP 服务器，提供 `get_weather(city: string)` 工具。这个服务器可以被 Claude Code SDK、Claude Desktop App、甚至其他 AI 应用调用。

### 16.3.3 步骤一：搭建 MCP 服务器骨架

```typescript
// file: mcp-servers/weather-server/index.ts

import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";

// 创建 MCP 服务器实例
const server = new Server(
  {
    name: "weather-server",
    version: "1.0.0",
  },
  {
    capabilities: {
      tools: {},
    },
  }
);

// TODO: 在下面注册 Tool 列表和 Tool 执行逻辑

// 启动服务器（使用 stdio 传输）
const transport = new StdioServerTransport();
await server.connect(transport);
```

### 16.3.4 步骤二：注册 Tool 列表

```typescript
// 注册 Tool 列表（客户端会先调用这个，知道你有哪些 Tool）
server.setRequestHandler(ListToolsRequestSchema, async () => {
  return {
    tools: [
      {
        name: "get_weather",
        description: "获取指定城市的当前天气。支持中国主要城市（北京、上海、广州、深圳...）。",
        inputSchema: {
          type: "object",
          properties: {
            city: {
              type: "string",
              description: "城市名称，例如：'北京'、'上海'"
            }
          },
          required: ["city"]
        }
      }
    ]
  };
});
```

### 16.3.5 步骤三：实现 Tool 执行逻辑

```typescript
// 模拟天气数据（实际项目中会调用天气 API）
const WEATHER_DATA: Record<string, { temp: number; condition: string }> = {
  "北京": { temp: 22, condition: "晴" },
  "上海": { temp: 25, condition: "多云" },
  "广州": { temp: 28, condition: "雷阵雨" },
  "深圳": { temp: 29, condition: "晴" }
};

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  if (request.params.name === "get_weather") {
    const city = request.params.arguments?.city as string;
    
    if (!city) {
      throw new Error("缺少必需参数：city");
    }
    
    const weather = WEATHER_DATA[city];
    
    if (!weather) {
      return {
        content: [{
          type: "text",
          text: `抱歉，暂时没有 ${city} 的天气数据。支持的城市：北京、上海、广州、深圳。`
        }]
      };
    }
    
    return {
      content: [{
        type: "text",
        text: `${city} 当前天气：${weather.condition}，温度 ${weather.temp}°C`
      }]
    };
  }
  
  throw new Error(`未知的 Tool：${request.params.name}`);
});
```

### 16.3.6 步骤四：在 Claude Code SDK 中调用 MCP 服务器

```typescript
// file: main-with-mcp.ts

import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { StdioClientTransport } from "@modelcontextprotocol/sdk/client/stdio.js";
import Anthropic from '@anthropic-ai/sdk';

// 启动 MCP 服务器（作为子进程）
const transport = new StdioClientTransport({
  command: "node",
  args: ["mcp-servers/weather-server/index.js"]
});

const mcpClient = new Client({
  name: "weather-client",
  version: "1.0.0"
}, { capabilities: {} });

await mcpClient.connect(transport);

// 获取 MCP 服务器提供的 Tool 列表
const toolsResult = await mcpClient.listTools();
const mcpTools = toolsResult.tools.map(tool => ({
  name: tool.name,
  description: tool.description,
  input_schema: tool.inputSchema
}));

// 在 Claude Code SDK 中使用这些 Tool
const anthropic = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY!
});

async function chatWithWeather(userMessage: string) {
  const response = await anthropic.messages.create({
    model: "claude-sonnet-4-20250514",
    max_tokens: 1024,
    tools: mcpTools,  // ← 使用 MCP 提供的 Tool
    messages: [{ role: "user", content: userMessage }]
  });
  
  // 处理 Tool 调用（和自定义 Tool 一样）
  for (const block of response.content) {
    if (block.type === "tool_use") {
      // 调用 MCP 服务器执行 Tool
      const result = await mcpClient.callTool({
        name: block.name,
        arguments: block.input
      });
      
      console.log(`Claude 调用了 ${block.name}，结果：`, result.content[0].text);
    }
  }
}

// 测试
chatWithWeather("北京今天天气怎么样？");
```

---

## 16.4 插件发布和分发

### 16.4.1 自定义 Tool