# 第5章：Message 与 Content 类型

> "消息是灵魂的载体，Content Block 是灵魂的形状。" —— 老三

如果说 Claude Code SDK 是一座大厦，那 Message 和 Content 就是砖和瓦。几乎所有 SDK 操作都围绕这两个核心类型展开：你发消息给 Claude，Claude 回消息给你；消息里装的是 Content Block，Content Block 决定了消息的"形状"——纯文本、图片、还是工具调用。

本章深入拆解 Message 和 Content 类型，让你彻底搞清楚数据是怎么流动的。

---

## 5.1 Message 结构详解

### 5.1.1 Message 的完整定义

一条消息（Message）是 Claude 对话中的基本单元。SDK 中，Message 的核心结构如下：

```typescript
interface Message {
  role: 'user' | 'assistant';  // 谁说的
  content: ContentBlock[];      // 说了什么（内容块数组）
}
```

看起来简单对吧？但 `ContentBlock[]` 是个万花筒——它可以是文字、图片、工具调用请求、工具执行结果，甚至是它们的混合。

### 5.1.2 User Message vs Assistant Message

**User Message**（用户消息）：你发给 Claude 的内容。

```typescript
const userMessage: Message = {
  role: 'user',
  content: [
    { type: 'text', text: '这段代码有什么问题？' },
    { type: 'image', source: { type: 'base64', media_type: 'image/png', data: '...' } },
  ],
};
```

**Assistant Message**（助手消息）：Claude 返回的内容。

```typescript
const assistantMessage: Message = {
  role: 'assistant',
  content: [
    { type: 'text', text: '这段代码有两个问题：' },
    { type: 'text', text: '1. 变量未声明\n2. 缺少返回值' },
    { type: 'tool_use', id: 'toolu_01', name: 'read_file', input: { path: '/src/index.ts' } },
  ],
};
```

**关键区别：**
- `user` 消息可以包含 `text`、`image` 类型的 Content Block
- `assistant` 消息可以包含 `text`、`tool_use` 类型的 Content Block
- `tool_result` 只能出现在 `user` 消息中（作为对之前 `tool_use` 的回应）

---

## 5.2 Content Block 类型全解

Content Block 是消息的原子单位。一条消息可以有多个 Content Block，组成丰富的多模态内容。

### 5.2.1 文本内容（Text Block）

最基础的类型，就是一段文字：

```typescript
interface TextBlock {
  type: 'text';
  text: string;
}
```

**示例：纯文本对话**

```typescript
import { ClaudeCode } from '@anthropic-ai/claude-code-sdk';

const claude = new ClaudeCode();

async function simpleTextChat() {
  const response = await claude.messages.create({
    model: 'claude-sonnet-4-20250514',
    maxTokens: 1024,
    messages: [
      {
        role: 'user',
        content: [
          { type: 'text', text: '用三行诗形容 TypeScript' },
        ],
      },
    ],
  });

  // 遍历所有文本内容块
  for (const block of response.content) {
    if (block.type === 'text') {
      console.log(block.text);
    }
  }
}

simpleTextChat();
```

**输出示例：**

```
类型如铠甲，
编译似修行，
any 是投降。
```

### 5.2.2 图片内容（Image Block）

Claude 支持识别图片内容。你可以发送 Base64 编码的图片或图片 URL：

```typescript
interface ImageBlock {
  type: 'image';
  source: {
    type: 'base64' | 'url';
    media_type: 'image/jpeg' | 'image/png' | 'image/gif' | 'image/webp';
    data: string;       // base64 编码的图片数据（type=base64 时）
    url?: string;       // 图片 URL（type=url 时）
  };
}
```

**示例：发送图片让 Claude 分析**

```typescript
import { ClaudeCode } from '@anthropic-ai/claude-code-sdk';
import * as fs from 'fs';

const claude = new ClaudeCode();

async function analyzeImage() {
  // 读取本地图片并转为 base64
  const imageBuffer = fs.readFileSync('./screenshot.png');
  const base64Image = imageBuffer.toString('base64');

  const response = await claude.messages.create({
    model: 'claude-sonnet-4-20250514',
    maxTokens: 2048,
    messages: [
      {
        role: 'user',
        content: [
          {
            type: 'image',
            source: {
              type: 'base64',
              media_type: 'image/png',
              data: base64Image,
            },
          },
          {
            type: 'text',
            text: '描述这张截图中的 UI 布局，并指出可能的可用性问题。',
          },
        ],
      },
    ],
  });

  console.log(response.content[0].text);
}

analyzeImage();
```

**也支持 URL 方式：**

```typescript
{
  type: 'image',
  source: {
    type: 'url',
    url: 'https://example.com/diagram.png',
  },
}
```

> **老三的Tips：** 图片大小上限约 5MB，支持的格式：JPEG、PNG、GIF、WebP。超大的图片会降低响应速度，建议压缩后发送。

### 5.2.3 工具调用请求（Tool Use Block）

当 Claude 决定调用一个工具时，响应中会出现 `tool_use` 类型的 Content Block：

```typescript
interface ToolUseBlock {
  type: 'tool_use';
  id: string;       // 唯一标识符，如 "toolu_01ABC"
  name: string;     // 工具名称
  input: object;    // 工具输入参数（JSON 对象）
}
```

**示例：Claude 请求调用工具**

```typescript
import { ClaudeCode } from '@anthropic-ai/claude-code-sdk';

const claude = new ClaudeCode();

async function toolUseExample() {
  const response = await claude.messages.create({
    model: 'claude-sonnet-4-20250514',
    maxTokens: 4096,
    tools: [
      {
        name: 'get_weather',
        description: '获取指定城市的天气信息',
        input_schema: {
          type: 'object' as const,
          properties: {
            city: { type: 'string', description: '城市名称' },
          },
          required: ['city'],
        },
      },
    ],
    messages: [
      {
        role: 'user',
        content: '北京今天天气怎么样？',
      },
    ],
  });

  // 检查响应中的内容块类型
  for (const block of response.content) {
    if (block.type === 'text') {
      console.log('文本回复：', block.text);
    } else if (block.type === 'tool_use') {
      console.log('工具调用：', block.name);
      console.log('调用参数：', JSON.stringify(block.input));
      console.log('调用 ID：', block.id);
    }
  }
}

toolUseExample();
```

**输出示例：**

```
工具调用：get_weather
调用参数：{"city":"北京"}
调用 ID：toolu_01ABC123
```

### 5.2.4 工具执行结果（Tool Result Block）

当 Claude 请求调用工具后，你需要把工具执行的结果作为 `tool_result` 回传给 Claude。这是 tool_use 的"回执"。

```typescript
interface ToolResultBlock {
  type: 'tool_result';
  tool_use_id: string;  // 对应的 tool_use ID
  content: string;       // 执行结果
  is_error?: boolean;    // 是否为错误结果
}
```

> **重要：** `tool_result` 必须放在 `role: 'user'` 的消息中，而不是 `role: 'assistant'`。这是因为从对话角度看，工具结果是你（用户侧）提供给 Claude 的信息。

---

## 5.3 工具调用的完整流程

把上面几种 Content Block 串起来，一个完整的工具调用流程是这样的：

```
用户提问 → Claude 返回 tool_use → 你执行工具 → 回传 tool_result → Claude 生成最终回复
```

### 5.3.1 完整可运行示例：天气查询工具

```typescript
import { ClaudeCode } from '@anthropic-ai/claude-code-sdk';

const claude = new ClaudeCode();

// 模拟天气 API
function getWeather(city: string): string {
  const weatherData: Record<string, string> = {
    '北京': '晴天，25°C，空气质量良好',
    '上海': '多云，28°C，有轻微雾霾',
    '深圳': '阵雨，30°C，湿度 85%',
  };
  return weatherData[city] || `${city}：暂无天气数据`;
}

async function chatWithTool() {
  // 第1步：用户提问
  const userQuestion = '北京和上海今天天气怎么样？应该穿什么？';

  // 第2步：发送给 Claude（带工具定义）
  const response1 = await claude.messages.create({
    model: 'claude-sonnet-4-20250514',
    maxTokens: 4096,
    tools: [
      {
        name: 'get_weather',
        description: '获取指定城市的当前天气信息',
        input_schema: {
          type: 'object' as const,
          properties: {
            city: { type: 'string', description: '城市名称' },
          },
          required: ['city'],
        },
      },
    ],
    messages: [
      { role: 'user', content: userQuestion },
    ],
  });

  // 第3步：收集 Claude 的工具调用请求
  const toolCalls = response1.content.filter(
    (block): block is Extract<typeof block, { type: 'tool_use' }> =>
      block.type === 'tool_use'
  );

  if (toolCalls.length === 0) {
    // Claude 没有调用工具，直接输出文本回复
    const textBlocks = response1.content.filter(b => b.type === 'text');
    console.log(textBlocks.map(b => b.text).join(''));
    return;
  }

  // 第4步：执行所有工具调用，生成 tool_result
  const toolResults = toolCalls.map((call) => {
    let result: string;
    switch (call.name) {
      case 'get_weather':
        result = getWeather(call.input.city as string);
        break;
      default:
        result = `未知工具：${call.name}`;
    }
    return {
      type: 'tool_result' as const,
      tool_use_id: call.id,
      content: result,
    };
  });

  console.log('--- 工具执行结果 ---');
  toolResults.forEach(r => console.log(r.content));
  console.log('--- Claude 的最终回复 ---');

  // 第5步：将工具结果回传给 Claude，获取最终回复
  const response2 = await claude.messages.create({
    model: 'claude-sonnet-4-20250514',
    maxTokens: 4096,
    tools: [
      {
        name: 'get_weather',
        description: '获取指定城市的当前天气信息',
        input_schema: {
          type: 'object' as const,
          properties: {
            city: { type: 'string', description: '城市名称' },
          },
          required: ['city'],
        },
      },
    ],
    messages: [
      { role: 'user', content: userQuestion },
      { role: 'assistant', content: response1.content },  // Claude 的 tool_use 回复
      { role: 'user', content: toolResults },              // 工具执行结果
    ],
  });

  // 输出最终回复
  for (const block of response2.content) {
    if (block.type === 'text') {
      console.log(block.text);
    }
  }
}

chatWithTool();
```

**输出示例：**

```
--- 工具执行结果 ---
晴天，25°C，空气质量良好
多云，28°C，有轻微雾霾
--- Claude 的最终回复 ---
根据查询结果：

北京今天晴天，25°C，空气质量良好。建议穿轻薄的长袖或短袖，可以带一件薄外套。

上海多云，28°C，有轻微雾霾。温度比北京稍高，穿短袖即可，但建议戴口罩应对雾霾。

总的来说，两个城市都比较温暖，不需要厚外套，但上海注意防雾霾。
```

### 5.3.2 流程图解

```
┌─────────┐     user message      ┌─────────┐
│   用户   │ ──────────────────→  │  Claude  │
└─────────┘                       └─────────┘
                                       │
                              assistant message
                              (包含 tool_use block)
                                       │
                                       ▼
                                  ┌──────────┐
                                  │  你的代码  │
                                  │ 执行工具   │
                                  └──────────┘
                                       │
                              user message
                              (包含 tool_result block)
                                       │
                                       ▼
                                  ┌─────────┐
                                  │  Claude  │
                                  │ 生成回复  │
                                  └─────────┘
```

---

## 5.4 Content Block 的混合使用

一条消息可以包含多个不同类型的 Content Block。这在实际应用中非常常见。

### 5.4.1 文本 + 图片混合

```typescript
const response = await claude.messages.create({
  model: 'claude-sonnet-4-20250514',
  maxTokens: 2048,
  messages: [
    {
      role: 'user',
      content: [
        {
          type: 'image',
          source: {
            type: 'base64',
            media_type: 'image/png',
            data: base64Screenshot,
          },
        },
        {
          type: 'text',
          text: '这是我的网页截图。请告诉我布局有什么问题，并给出 CSS 修改建议。',
        },
      ],
    },
  ],
});
```

### 5.4.2 Assistant 消息中的混合内容

Claude 的回复也可能同时包含文本和工具调用：

```typescript
// Claude 的回复可能长这样：
const assistantContent = [
  { type: 'text', text: '让我先查看一下项目的文件结构。' },
  { type: 'tool_use', id: 'toolu_01', name: 'list_files', input: { path: './src' } },
  { type: 'tool_use', id: 'toolu_02', name: 'read_file', input: { path: './package.json' } },
];

// 它在"思考"的同时就发起了多个工具调用，不用等你提示
```

### 5.4.3 多轮对话的消息历史

多轮对话就是把每轮的 Message 依次追加到 `messages` 数组里：

```typescript
const messages: Message[] = [];

// 第1轮
messages.push({ role: 'user', content: '什么是闭包？' });
const resp1 = await claude.messages.create({ model: '...', maxTokens: 1024, messages });
messages.push({ role: 'assistant', content: resp1.content });

// 第2轮
messages.push({ role: 'user', content: '能举一个 TypeScript 的例子吗？' });
const resp2 = await claude.messages.create({ model: '...', maxTokens: 1024, messages });
messages.push({ role: 'assistant', content: resp2.content });

// 第3轮——Claude 记得之前所有的上下文
messages.push({ role: 'user', content: '那这个例子和 React 的 Hook 有什么关系？' });
const resp3 = await claude.messages.create({ model: '...', maxTokens: 1024, messages });
```

> **老三的Tips：** 每次调用 `messages.create()` 都要把完整的对话历史传进去。SDK 本身是无状态的——它不会帮你记住上一次的对话。所以你要自己维护 `messages` 数组。

---

## 5.5 实战：Content Block 类型守卫

在处理 Claude 的响应时，你需要判断每个 Content Block 的类型。TypeScript 的类型守卫（Type Guard）是完美的工具：

```typescript
import { ClaudeCode } from '@anthropic-ai/claude-code-sdk';

const claude = new ClaudeCode();

// 类型守卫函数
function isTextBlock(block: any): block is { type: 'text'; text: string } {
  return block.type === 'text';
}

function isToolUseBlock(block: any): block is { type: 'tool_use'; id: string; name: string; input: any } {
  return block.type === 'tool_use';
}

function isToolResultBlock(block: any): block is { type: 'tool_result'; tool_use_id: string; content: string } {
  return block.type === 'tool_result';
}

async function processResponse(response: any) {
  for (const block of response.content) {
    if (isTextBlock(block)) {
      console.log('📝 文本：', block.text);
    } else if (isToolUseBlock(block)) {
      console.log('🔧 工具调用：', block.name, block.input);
    } else if (isToolResultBlock(block)) {
      console.log('📋 工具结果：', block.content);
    } else {
      console.log('❓ 未知类型：', block.type);
    }
  }
}
```

### 5.5.1 完整示例：内容块路由器

一个更实用的模式——根据 Content Block 类型分发到不同的处理器：

```typescript
#!/usr/bin/env node
import { ClaudeCode } from '@anthropic-ai/claude-code-sdk';

const claude = new ClaudeCode();

// 定义处理器接口
interface ContentHandler<T> {
  canHandle(block: any): block is T;
  handle(block: T): Promise<void>;
}

// 文本处理器
const textHandler: ContentHandler<{ type: 'text'; text: string }> = {
  canHandle: (block): block is { type: 'text'; text: string } => block.type === 'text',
  handle: async (block) => {
    console.log('💬', block.text);
  },
};

// 工具调用处理器
const toolUseHandler: ContentHandler<{ type: 'tool_use'; id: string; name: string; input: any }> = {
  canHandle: (block) => block.type === 'tool_use',
  handle: async (block) => {
    console.log(`🔧 调用 ${block.name}(${JSON.stringify(block.input)}) [id=${block.id}]`);
  },
};

// 注册所有处理器
const handlers: ContentHandler<any>[] = [textHandler, toolUseHandler];

async function routeContent(content: any[]) {
  for (const block of content) {
    const handler = handlers.find(h => h.canHandle(block));
    if (handler) {
      await handler.handle(block);
    } else {
      console.log(`⚠️ 未处理的块类型: ${block.type}`);
    }
  }
}

// 测试
async function main() {
  const response = await claude.messages.create({
    model: 'claude-sonnet-4-20250514',
    maxTokens: 1024,
    messages: [{ role: 'user', content: '你好，请简单介绍你自己。' }],
  });

  await routeContent(response.content);
}

main().catch(console.error);
```

---

## 5.6 常见陷阱与最佳实践

### 5.6.1 不要假设 content 只有一个块

```typescript
// ❌ 错误：假设只有一个文本块
const text = response.content[0].text;

// ✅ 正确：遍历所有内容块
for (const block of response.content) {
  if (block.type === 'text') {
    console.log(block.text);
  }
}
```

Claude 可能把一段回复拆成多个文本块，或者在文本之间插入工具调用。始终用循环处理。

### 5.6.2 tool_result 必须与 tool_use 配对

```typescript
// ❌ 错误：tool_result 的 ID 不匹配
{ type: 'tool_result', tool_use_id: 'wrong_id', content: '...' }

// ✅ 正确：使用 Claude 返回的 tool_use.id
{ type: 'tool_result', tool_use_id: toolUseBlock.id, content: '...' }
```

### 5.6.3 消息顺序必须严格交替

```typescript
// ❌ 错误：两条 user 消息连续出现
messages: [
  { role: 'user', content: '问题1' },
  { role: 'user', content: '问题2' },  // 不允许！
]

// ✅ 正确：先处理第一条，拿到 assistant 回复，再问第二条
messages: [
  { role: 'user', content: '问题1' },
  { role: 'assistant', content: resp1.content },
  { role: 'user', content: '问题2' },
]
```

> **注意：** `tool_result` 放在 `user` 消息中，所以可能出现连续两条 `user` 消息（一条是工具结果，一条是新问题）。但 `user` → `user` 只在工具调用场景下被允许。一般的对话仍需严格交替。

### 5.6.4 图片要控制大小

```typescript
// ❌ 发送原始高清截图（可能 10MB+）
const raw = fs.readFileSync('./huge-screenshot.png');
const base64 = raw.toString('base64');

// ✅ 先压缩，再发送
import sharp from 'sharp';

const compressed = await sharp('./huge-screenshot.png')
  .resize(1600)       // 限制最大宽度
  .jpeg({ quality: 80 })  // 转为 JPEG
  .toBuffer();

const base64 = compressed.toString('base64');
```

---

## 5.7 小结

本章深入拆解了 Claude Code SDK 中最核心的两个类型：

1. **Message** - 对话的基本单元，由 `role`（user/assistant）和 `content`（ContentBlock 数组）组成
2. **Content Block** - 消息的原子单位，有四种类型：
   - `text` - 文本内容
   - `image` - 图片内容（Base64 或 URL）
   - `tool_use` - 工具调用请求（Claude 发起）
   - `tool_result` - 工具执行结果（你回传）
3. **工具调用流程** - user → assistant(tool_use) → user(tool_result) → assistant(最终回复)
4. **最佳实践** - 遍历 content、配对 ID、交替消息、控制图片大小

掌握这些，你就掌握了 Claude Code SDK 的数据骨架。下一章我们将深入 Tool 定义与调用，看看如何让 Claude 使用你定义的工具。

---

## 练习题

1. **图片分析器**：写一个程序，接受本地图片路径作为命令行参数，让 Claude 分析图片内容并输出文字描述。

2. **多工具对话**：定义 3 个工具（如 `get_time`、`calculate`、`search`），实现一个完整的工具调用循环——用户提问、Claude 调用工具、你回传结果、Claude 生成回复。

3. **挑战题**：实现一个"工具调用链"——Claude 在一次回复中可能调用多个工具，你的代码需要并发执行所有工具调用，收集结果后一次性回传。

---

> **老三的Tips：**
> - 搞不清楚 Content Block 的类型？打印 `JSON.stringify(response, null, 2)` 看完整结构
> - 工具调用总是"一问一答"模式：Claude 请求 → 你执行 → 回传结果 → Claude 继续
> - 多个工具调用可以并发执行，但 `tool_result` 必须每个都回传

---

_本章完，下一章：Tool 定义与调用 ➡️_
