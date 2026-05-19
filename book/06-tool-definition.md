# 第6章：Tool 定义与调用

> "给 Claude 一把锤子，它不仅能帮你敲钉子，还会告诉你螺丝刀在哪儿。" —— 老三

在第5章，我们初步接触了 `tool_use` 和 `tool_result` 这两种 Content Block。但那是"看个热闹"——这一章，我们要"看门道"：如何定义 Tool、如何让 Claude 精准调用、如何处理工具执行结果、以及如何构建复杂的工具链。

Tool（工具）是 Claude Code SDK 最强大的功能之一。它让 Claude 从"聊天机器人"变成"能动手的助手"——可以查天气、读文件、调 API、跑代码……几乎没有做不到，只有你没定义的。

---

## 6.1 Tool 的定义：Schema 是灵魂

### 6.1.1 Tool 的完整结构

在 Claude Code SDK 中，一个 Tool 的定义包含三个核心字段：

```typescript
interface Tool {
  name: string;           // 工具名称（字母、数字、下划线，如 "get_weather"）
  description: string;    // 工具描述（Claude 靠这个决定"要不要调用"）
  input_schema: {         // 参数定义（JSON Schema 格式）
    type: 'object';
    properties: Record<string, object>;
    required?: string[];
  };
}
```

**重点：** `input_schema` 是标准的 [JSON Schema](https://json-schema.org/) 格式，Claude 会根据这个 Schema 生成合法的调用参数。写错一个字段，Claude 就可能调用失败或参数不对。

### 6.1.2 最简单的 Tool：无参数

有些工具不需要任何参数，比如"获取当前时间"：

```typescript
const getTimeTool = {
  name: 'get_current_time',
  description: '获取当前的日期和时间（ISO 8601 格式）',
  input_schema: {
    type: 'object' as const,
    properties: {},  // 没有参数
    required: [],
  },
};
```

**示例：调用无参数工具**

```typescript
import { ClaudeCode } from '@anthropic-ai/claude-code-sdk';

const claude = new ClaudeCode();

async function callNoArgTool() {
  const response = await claude.messages.create({
    model: 'claude-sonnet-4-20250514',
    maxTokens: 1024,
    tools: [getTimeTool],
    messages: [
      { role: 'user', content: '现在几点了？' },
    ],
  });

  // 检查 Claude 是否调用了工具
  const toolUse = response.content.find(b => b.type === 'tool_use');
  if (toolUse) {
    console.log('Claude 调用了工具：', toolUse.name);
    console.log('调用 ID：', toolUse.id);
    console.log('输入参数：', toolUse.input);  // 应该是 {}
  }
}

const getTimeTool = {
  name: 'get_current_time',
  description: '获取当前的日期和时间（ISO 8601 格式）',
  input_schema: {
    type: 'object' as const,
    properties: {},
    required: [],
  },
};

callNoArgTool().catch(console.error);
```

### 6.1.3 带参数的 Tool：JSON Schema 详解

大多数工具需要参数。你用 JSON Schema 定义参数类型、描述、是否必填。

**示例：天气查询工具（一个必填参数）**

```typescript
const getWeatherTool = {
  name: 'get_weather',
  description: '获取指定城市的当前天气信息。支持中国主要城市。',
  input_schema: {
    type: 'object' as const,
    properties: {
      city: {
        type: 'string',
        description: '城市名称，如"北京"、"上海"、"深圳"',
      },
      unit: {
        type: 'string',
        enum: ['celsius', 'fahrenheit'],
        description: '温度单位，默认为 celsius（摄氏度）',
      },
    },
    required: ['city'],  // city 是必填的，unit 是可选的
  },
};
```

**关键点：**
- `properties` 里定义的每个字段，Claude 都可能填充
- `required` 数组里的字段，Claude **必须**提供（否则 Claude 会拒绝调用这个工具）
- `description` 很重要！Claude 靠描述理解"这个参数是干什么的"
- `enum` 可以限制参数的取值范围，Claude 会从枚举值中选择

### 6.1.4 复杂参数：嵌套对象与数组

工具参数可以是嵌套的对象或数组，只要 JSON Schema 支持就行。

**示例：代码搜索工具（嵌套对象参数）**

```typescript
const searchCodeTool = {
  name: 'search_code',
  description: '在代码仓库中搜索符合特定条件的代码片段',
  input_schema: {
    type: 'object' as const,
    properties: {
      query: {
        type: 'string',
        description: '搜索关键词，支持正则表达式',
      },
      options: {
        type: 'object',
        description: '搜索选项',
        properties: {
          case_sensitive: {
            type: 'boolean',
            description: '是否区分大小写，默认 false',
          },
          file_types: {
            type: 'array',
            items: { type: 'string' },
            description: '限制搜索的文件类型，如 [".ts", ".js"]',
          },
          max_results: {
            type: 'number',
            description: '最多返回结果数，默认 10',
            minimum: 1,
            maximum: 100,
          },
        },
      },
    },
    required: ['query'],
  },
};
```

Claude 调用时可能生成这样的 `input`：

```json
{
  "query": "function.*Claude",
  "options": {
    "case_sensitive": false,
    "file_types": [".ts", ".js"],
    "max_results": 20
  }
}
```

> **老三的Tips：** JSON Schema 很强大，但也很容易写错。建议先用 [JSON Schema 在线编辑器](https://jsonschema.net/) 写好、验证，再复制到代码里。

---

## 6.2 Tool 调用完整流程

定义好工具后，完整的调用流程是：**定义工具 → 发送消息 → Claude 决定调用 → 你执行工具 → 回传结果 → Claude 继续**。

### 6.2.1 流程图解

```
┌─────────────┐
│  第1步：定义工具（你写代码）              │
│  tools: [{ name, description, input_schema }] │
└─────────────┘
                    ↓
┌─────────────┐
│  第2步：发送消息（带 tools 参数）        │
│  claude.messages.create({ tools, messages })   │
└─────────────┘
                    ↓
┌─────────────┐
│  第3步：Claude 决定是否调用工具          │
│  如果不需要工具，直接返回文本            │
│  如果需要工具，返回 tool_use content     │
└─────────────┘
                    ↓
┌─────────────┐
│  第4步：你执行工具（你写代码）          │
│  解析 tool_use.input，执行实际逻辑       │
└─────────────┘
                    ↓
┌─────────────┐
│  第5步：回传工具结果（tool_result）     │
│  messages.push({ role: 'user', content:   │
│    [{ type: 'tool_result', tool_use_id,   │
│     content: '执行结果' }] })            │
└─────────────┘
                    ↓
┌─────────────┐
│  第6步：Claude 生成最终回复             │
│  基于工具执行结果，生成用户友好的回答   │
└─────────────┘
```

### 6.2.2 完整可运行示例：天气查询

这个例子把整个流程串起来，每一步都有详细注释：

```typescript
#!/usr/bin/env node
import { ClaudeCode } from '@anthropic-ai/claude-code-sdk';

const claude = new ClaudeCode();

// ========== 第1步：定义工具 ==========
const getWeatherTool = {
  name: 'get_weather',
  description: '获取指定城市的当前天气信息。支持中国主要城市。',
  input_schema: {
    type: 'object' as const,
    properties: {
      city: {
        type: 'string',
        description: '城市名称，如"北京"、"上海"、"深圳"',
      },
    },
    required: ['city'],
  },
};

// ========== 模拟工具执行（实际项目中可能是 API 调用） ==========
function executeGetWeather(city: string): string {
  const weatherDB: Record<string, string> = {
    '北京': '晴天，25°C，空气质量：良',
    '上海': '多云，28°C，空气质量：轻度污染',
    '深圳': '阵雨，30°C，湿度 85%',
    '广州': '雷阵雨，29°C，湿度 80%',
  };

  if (city in weatherDB) {
    return weatherDB[city];
  } else {
    return `抱歉，暂无 ${city} 的天气数据。`;
  }
}

// ========== 主函数 ==========
async function weatherChat(userQuestion: string) {
  const messages: any[] = [];

  // ========== 第2步：发送消息（带 tools） ==========
  messages.push({ role: 'user', content: userQuestion });

  let response = await claude.messages.create({
    model: 'claude-sonnet-4-20250514',
    maxTokens: 2048,
    tools: [getWeatherTool],
    messages: messages,
  });

  // 把 Claude 的回复加入历史
  messages.push({ role: 'assistant', content: response.content });

  // ========== 第3步：检查 Claude 是否调用了工具 ==========
  const toolUses = response.content.filter(
    (block: any) => block.type === 'tool_use'
  );

  if (toolUses.length === 0) {
    // Claude 没有调用工具，直接输出文本回复
    const text = response.content
      .filter((b: any) => b.type === 'text')
      .map((b: any) => b.text)
      .join('');
    console.log('Claude 回复：', text);
    return;
  }

  // ========== 第4步：执行所有工具调用 ==========
  const toolResults = toolUses.map((toolUse: any) => {
    console.log(`🔧 执行工具：${toolUse.name}`);
    console.log(`   参数：`, toolUse.input);

    let result: string;
    if (toolUse.name === 'get_weather') {
      result = executeGetWeather(toolUse.input.city);
    } else {
      result = `未知工具：${toolUse.name}`;
    }

    console.log(`   结果：`, result);

    // ========== 第5步：构造 tool_result ==========
    return {
      type: 'tool_result' as const,
      tool_use_id: toolUse.id,
      content: result,
    };
  });

  // 把工具结果加入消息历史
  messages.push({ role: 'user', content: toolResults });

  // ========== 第6步：让 Claude 生成最终回复 ==========
  response = await claude.messages.create({
    model: 'claude-sonnet-4-20250514',
    maxTokens: 2048,
    tools: [getWeatherTool],
    messages: messages,
  });

  // 输出最终回复
  const finalText = response.content
    .filter((b: any) => b.type === 'text')
    .map((b: any) => b.text)
    .join('');

  console.log('--- Claude 的最终回复 ---');
  console.log(finalText);
}

// 运行测试
weatherChat('北京和深圳今天天气怎么样？适合穿什么衣服？')
  .catch(console.error);
```

**运行结果示例：**

```
🔧 执行工具：get_weather
   参数： { city: '北京' }
   结果： 晴天，25°C，空气质量：良
🔧 执行工具：get_weather
   参数： { city: '深圳' }
   结果： 阵雨，30°C，湿度 85%
--- Claude 的最终回复 ---
根据查询结果：

北京今天晴天，25°C，空气质量良好。建议穿轻薄长袖或短袖，可以带件薄外套备用。

深圳今天有阵雨，30°C，湿度较高（85%）。建议穿短袖+防水外套，记得带伞。

总体来说，深圳温度更高但下雨，北京天气更好但稍凉一些。
```

---

## 6.3 处理 Tool Result：成功与错误

工具执行可能成功，也可能失败。你需要把这两种情况都告诉 Claude。

### 6.3.1 成功结果

成功时，直接把结果放在 `content` 里：

```typescript
const successResult = {
  type: 'tool_result' as const,
  tool_use_id: toolUse.id,
  content: '晴天，25°C，空气质量：良',
};
```

### 6.3.2 错误结果

如果工具执行失败（如网络错误、参数非法），设置 `is_error: true`：

```typescript
const errorResult = {
  type: 'tool_result' as const,
  tool_use_id: toolUse.id,
  content: '错误：无法连接到天气 API（网络超时）',
  is_error: true,
};
```

**Claude 看到 `is_error: true` 后，会：**
1. 知道工具执行失败了
2. 可能在回复中解释错误原因
3. 或者尝试用其他方式回答用户问题

### 6.3.3 完整示例：带错误处理的工具执行

```typescript
#!/usr/bin/env node
import { ClaudeCode } from '@anthropic-ai/claude-code-sdk';

const claude = new ClaudeCode();

const calcTool = {
  name: 'calculate',
  description: '执行数学计算。支持加减乘除和括号。',
  input_schema: {
    type: 'object' as const,
    properties: {
      expression: {
        type: 'string',
        description: '数学表达式，如 "2 + 3 * 4" 或 "(10 - 3) / 2"',
      },
    },
    required: ['expression'],
  },
};

// 安全的数学计算器（防止代码注入）
function safeCalculate(expr: string): { result: number | null; error: string | null } {
  // 只允许数字、加减乘除、括号、空格
  const allowedPattern = /^[\d\s\+\-\*\/\(\)\.]+$/;

  if (!allowedPattern.test(expr)) {
    return { result: null, error: '非法表达式：包含不允许的字符' };
  }

  try {
    // 使用 Function 构造函数（比 eval 稍微安全一点）
    const result = new Function('return ' + expr)();
    return { result, error: null };
  } catch (e) {
    return { result: null, error: `计算错误：${(e as Error).message}` };
  }
}

async function calcChat(question: string) {
  const messages: any[] = [
    { role: 'user', content: question },
  ];

  let response = await claude.messages.create({
    model: 'claude-sonnet-4-20250514',
    maxTokens: 1024,
    tools: [calcTool],
    messages: messages,
  });

  messages.push({ role: 'assistant', content: response.content });

  // 处理所有 tool_use
  const toolUses = response.content.filter((b: any) => b.type === 'tool_use');

  if (toolUses.length > 0) {
    const toolResults = toolUses.map((toolUse: any) => {
      if (toolUse.name === 'calculate') {
        const { result, error } = safeCalculate(toolUse.input.expression);

        if (error) {
          // 返回错误结果
          return {
            type: 'tool_result' as const,
            tool_use_id: toolUse.id,
            content: error,
            is_error: true,
          };
        } else {
          // 返回成功结果
          return {
            type: 'tool_result' as const,
            tool_use_id: toolUse.id,
            content: `计算结果：${result}`,
          };
        }
      }
    });

    messages.push({ role: 'user', content: toolResults });

    // 获取 Claude 的最终回复
    response = await claude.messages.create({
      model: 'claude-sonnet-4-20250514',
      maxTokens: 1024,
      tools: [calcTool],
      messages: messages,
    });
  }

  // 输出回复
  const text = response.content
    .filter((b: any) => b.type === 'text')
    .map((b: any) => b.text)
    .join('');

  console.log(text);
}

// 测试
calcChat('计算 (10 + 5) * 3 - 8 等于多少？')
  .catch(console.error);
```

**运行结果：**

```
(10 + 5) * 3 - 8 = 37

计算步骤：
1. 先算括号：10 + 5 = 15
2. 再乘：15 * 3 = 45
3. 最后减：45 - 8 = 37

所以答案是 37。
```

---

## 6.4 多工具并发调用

Claude 可以在一次回复中调用多个工具。你需要并发执行它们，然后一次性回传所有结果。

### 6.4.1 为什么需要并发？

假设用户问："北京和上海的天气分别是多少？" 

Claude 会同时调用两个 `get_weather` 工具。如果你串行执行（先查北京，再查上海），会浪费时间。正确做法是并发执行：

```typescript
// 并发执行所有工具调用
const toolResults = await Promise.all(
  toolUses.map(async (toolUse) => {
    switch (toolUse.name) {
      case 'get_weather':
        return await fetchWeather(toolUse.input.city);  // 异步获取
      case 'get_time':
        return await fetchTime(toolUse.input.timezone);
      default:
        return { tool_use_id: toolUse.id, content: '未知工具', is_error: true };
    }
  })
);
```

### 6.4.2 完整示例：并发查询多城市天气

```typescript
#!/usr/bin/env node
import { ClaudeCode } from '@anthropic-ai/claude-code-sdk';

const claude = new ClaudeCode();

// 定义天气工具
const getWeatherTool = {
  name: 'get_weather',
  description: '获取指定城市的当前天气信息',
  input_schema: {
    type: 'object' as const,
    properties: {
      city: {
        type: 'string',
        description: '城市名称',
      },
    },
    required: ['city'],
  },
};

// 模拟天气 API（带随机延迟）
async function fetchWeather(city: string): Promise<string> {
  const db: Record<string, string> = {
    '北京': '晴天，25°C',
    '上海': '多云，28°C',
    '深圳': '阵雨，30°C',
    '广州': '雷阵雨，29°C',
    '杭州': '阴天，26°C',
  };

  // 模拟网络延迟（100-500ms）
  await new Promise(resolve => setTimeout(resolve, 100 + Math.random() * 400));

  return db[city] || `暂无 ${city} 的数据`;
}

// 并发执行工具
async function executeToolsConcurrently(toolUses: any[]) {
  console.log(`📡 准备并发执行 ${toolUses.length} 个工具调用...`);

  const startTime = Date.now();

  // Promise.all 并发执行所有工具
  const results = await Promise.all(
    toolUses.map(async (toolUse) => {
      console.log(`  → 执行 ${toolUse.name}(${JSON.stringify(toolUse.input)})`);

      let result: string;
      if (toolUse.name === 'get_weather') {
        result = await fetchWeather(toolUse.input.city);  // 异步获取
      } else {
        result = `未知工具：${toolUse.name}`;
      }

      console.log(`  ← 返回：${result}`);

      return {
        type: 'tool_result' as const,
        tool_use_id: toolUse.id,
        content: result,
      };
    })
  );

  const elapsed = Date.now() - startTime;
  console.log(`⏱️  并发执行完成，耗时 ${elapsed}ms`);

  return results;
}

async function multiCityWeatherChat(question: string) {
  const messages: any[] = [
    { role: 'user', content: question },
  ];

  let response = await claude.messages.create({
    model: 'claude-sonnet-4-20250514',
    maxTokens: 2048,
    tools: [getWeatherTool],
    messages: messages,
  });

  messages.push({ role: 'assistant', content: response.content });

  const toolUses = response.content.filter((b: any) => b.type === 'tool_use');

  if (toolUses.length > 0) {
    // 并发执行
    const toolResults = await executeToolsConcurrently(toolUses);
    messages.push({ role: 'user', content: toolResults });

    response = await claude.messages.create({
      model: 'claude-sonnet-4-20250514',
      maxTokens: 2048,
      tools: [getWeatherTool],
      messages: messages,
    });
  }

  console.log('\n=== Claude 的回复 ===');
  response.content
    .filter((b: any) => b.type === 'text')
    .forEach((b: any) => console.log(b.text));
}

// 测试并发
multiCityWeatherChat('北京、上海、深圳今天的天气分别是多少？')
  .catch(console.error);
```

**输出示例：**

```
📡 准备并发执行 3 个工具调用...
  → 执行 get_weather({"city":"北京"})
  → 执行 get_weather({"city":"上海"})
  → 执行 get_weather({"city":"深圳"})
  ← 返回：晴天，25°C
  ← 返回：多云，28°C
  ← 返回：阵雨，30°C
⏱️  并发执行完成，耗时 412ms

=== Claude 的回复 ===
北京、上海、深圳今天的天气如下：

| 城市 | 天气 | 温度 |
|------|------|------|
| 北京 | 晴天 | 25°C |
| 上海 | 多云 | 28°C |
| 深圳 | 阵雨 | 30°C |

深圳今天有阵雨，出门记得带伞！北京天气最好，适合户外活动。
```

**关键点：** 如果是串行执行，3 个工具各花 300-500ms，总共需要 900ms-1500ms；并发执行只需要等待最慢的那个（约 500ms），快了 2-3 倍。

---

## 6.5 Tool 的工具箱：常用模式集合

### 6.5.1 布尔类型参数

```typescript
{
  name: 'search',
  description: '搜索文件内容',
  input_schema: {
    type: 'object',
    properties: {
      keyword: { type: 'string', description: '搜索关键词' },
      case_sensitive: { type: 'boolean', description: '是否区分大小写，默认 false' },
      recursive: { type: 'boolean', description: '是否递归搜索子目录，默认 true' },
    },
    required: ['keyword'],
  },
}
```

### 6.5.2 枚举类型参数

```typescript
{
  name: 'get_weather',
  input_schema: {
    type: 'object',
    properties: {
      city: { type: 'string', description: '城市名称' },
      unit: {
        type: 'string',
        enum: ['celsius', 'fahrenheit', 'kelvin'],
        description: '温度单位',
      },
    },
    required: ['city'],
  },
}
```

### 6.5.3 可选参数 + 有默认值

```typescript
{
  name: 'create_file',
  input_schema: {
    type: 'object',
    properties: {
      path: { type: 'string', description: '文件路径' },
      content: { type: 'string', description: '文件内容' },
      encoding: {
        type: 'string',
        default: 'utf-8',      // 默认值
        description: '字符编码',
      },
      mode: {
        type: 'string',
        default: '0644',       // 默认值
        description: '文件权限（八进制）',
      },
    },
    required: ['path', 'content'],
  },
}
```

### 6.5.4 数组类型参数

```typescript
{
  name: 'batch_search',
  input_schema: {
    type: 'object',
    properties: {
      queries: {
        type: 'array',
        items: { type: 'string' },
        description: '搜索关键词列表',
        maxItems: 10,  // 最多 10 个
      },
      sources: {
        type: 'array',
        items: { type: 'string', enum: ['web', 'news', 'wiki'] },
        description: '搜索来源',
      },
    },
    required: ['queries'],
  },
}
```

---

## 6.6 工具注册与管理

### 6.6.1 集中管理工具列表

随着工具数量增加，建议集中管理工具定义：

```typescript
// tools/index.ts
import { Tool } from '@anthropic-ai/claude-code-sdk';

// 工具定义
export const tools = {
  get_weather: {
    name: 'get_weather',
    description: '获取指定城市的天气信息',
    input_schema: { ... },
  },
  get_time: {
    name: 'get_time',
    description: '获取当前时间',
    input_schema: {
      type: 'object' as const,
      properties: {},
      required: [],
    },
  },
  calculate: {
    name: 'calculate',
    description: '执行数学计算',
    input_schema: { ... },
  },
} as const;

// 获取所有工具的数组（供 SDK 使用）
export const allTools = Object.values(tools);

// 执行工具的映射表
export const toolHandlers: Record<string, (input: any) => any> = {
  get_weather: (input) => fetchWeather(input.city),
  get_time: () => new Date().toISOString(),
  calculate: (input) => safeCalculate(input.expression),
};
```

### 6.6.2 动态注册工具

有时你需要根据配置动态注册工具：

```typescript
import { ClaudeCode } from '@anthropic-ai/claude-code-sdk';

const claude = new ClaudeCode();

interface ToolConfig {
  name: string;
  description: string;
  handler: (input: any) => Promise<string>;
  schema: any;
}

// 模拟：从配置文件加载工具
const toolConfigs: ToolConfig[] = [
  {
    name: 'get_weather',
    description: '获取天气',
    handler: async (input) => `北京：晴天，25°C`,
    schema: { type: 'object', properties: { city: { type: 'string' } }, required: ['city'] },
  },
  {
    name: 'get_news',
    description: '获取新闻',
    handler: async (input) => `今日头条：...`,
    schema: { type: 'object', properties: {}, required: [] },
  },
];

// 动态生成工具列表
const dynamicTools = toolConfigs.map(config => ({
  name: config.name,
  description: config.description,
  input_schema: config.schema,
}));

// 动态生成处理器
const handlerMap = new Map(
  toolConfigs.map(config => [config.name, config.handler])
);

// 执行工具
async function executeTool(toolUse: any) {
  const handler = handlerMap.get(toolUse.name);
  if (!handler) {
    return { tool_use_id: toolUse.id, content: `未知工具：${toolUse.name}`, is_error: true };
  }

  try {
    const result = await handler(toolUse.input);
    return { type: 'tool_result' as const, tool_use_id: toolUse.id, content: result };
  } catch (e) {
    return { type: 'tool_result' as const, tool_use_id: toolUse.id, content: `执行错误：${e}`, is_error: true };
  }
}

// 使用
async function dynamicToolDemo() {
  const response = await claude.messages.create({
    model: 'claude-sonnet-4-20250514',
    maxTokens: 1024,
    tools: dynamicTools,
    messages: [{ role: 'user', content: '今天的天气怎么样？' }],
  });

  const toolUses = response.content.filter((b: any) => b.type === 'tool_use');
  const results = await Promise.all(toolUses.map(executeTool));

  console.log(results);
}

dynamicToolDemo().catch(console.error);
```

---

## 6.7 常见陷阱与最佳实践

### 6.7.1 工具描述要清晰，但不要过长

```typescript
// ❌ 过于简略（Claude 可能不知道什么时候该用）
{ name: 'calc', description: '计算' }

// ✅ 清晰描述用途和示例
{
  name: 'calculate',
  description: '执行数学计算。支持加减乘除、括号、指数运算。' +
               '示例：calculate("2 + 3 * 4") 返回 14',
}

// ❌ 过于冗长（Claude 可能忽略或困惑）
{
  name: 'calculate',
  description: '这是一个计算器工具，用于执行各种数学运算，包括但不限于：加法（用 + 表示）、减法（用 - 表示）、乘法（用 * 表示）、除法（用 / 表示）...'
}
```

### 6.7.2 参数描述要具体

```typescript
// ❌ 太模糊
properties: {
  query: { type: 'string' }  // 什么查询？
}

// ✅ 明确用途和格式
properties: {
  query: {
    type: 'string',
    description: '搜索关键词，支持空格分隔的多个词（如 "react hooks"）'
  }
}
```

### 6.7.3 处理 Claude 没有调用工具的情况

Claude 不是每次都会调用工具。当问题不需要外部数据时，它会直接回答：

```typescript
// 检查是否有工具调用
const toolUses = response.content.filter(b => b.type === 'tool_use');

if (toolUses.length === 0) {
  // Claude 没有调用工具，直接输出文本
  const text = response.content
    .filter(b => b.type === 'text')
    .map(b => b.text)
    .join('');
  console.log('Claude 回复：', text);
} else {
  // 有工具调用，正常处理...
}
```

### 6.7.4 工具超时处理

```typescript
async function executeToolWithTimeout(toolUse: any, timeoutMs = 10000) {
  const handler = toolHandlers[toolUse.name];

  return Promise.race([
    handler(toolUse.input),
    new Promise((_, reject) =>
      setTimeout(() => reject(new Error('工具执行超时')), timeoutMs)
    ),
  ]).catch((e) => ({
    type: 'tool_result' as const,
    tool_use_id: toolUse.id,
    content: `执行超时：${e.message}`,
    is_error: true,
  }));
}
```

---

## 6.8 小结

本章深入讲解了 Tool 的定义与调用：

1. **Tool 定义** - 由 `name`、`description`、`input_schema` 三部分组成，`input_schema` 使用 JSON Schema 格式
2. **完整调用流程** - 定义工具 → 发送消息 → Claude 判断调用 → 执行工具 → 回传结果 → Claude 生成回复
3. **处理工具结果** - 成功返回结果字符串，失败设置 `is_error: true`
4. **并发执行** - 用 `Promise.all` 并发执行多个工具调用，节省时间
5. **常用工具模式** - 布尔参数、枚举参数、可选参数、数组参数
6. **工具管理** - 集中管理工具列表、动态注册、处理器映射
7. **最佳实践** - 清晰的描述、具体的参数、超时处理

掌握这些，你就能让 Claude 像一个真正的助手一样——不仅能回答问题，还能动手做事。

---

## 练习题

1. **自定义工具**：定义一个 `translate` 工具，接受 `text`、`source_lang`、`target_lang` 三个参数，实现一个简单的翻译逻辑（可以用字典模拟）。

2. **并发抓取**：定义 `get_news` 工具让它并发抓取多个网站的新闻标题，统计总耗时，对比串行执行的耗时。

3. **工具链**：实现一个"代码执行链"——用户说"运行这个代码"，Claude 先调用 `validate_code` 工具验证代码安全性（检查是否有危险操作），通过后再调用 `execute_code` 工具真正执行。

4. **挑战题**：实现一个"自适应工具选择"功能——根据用户问题，Claude 自动决定调用哪些工具组合，实现一个复杂任务。

---

> **老三的Tips：**
> - `description` 是 Claude 决定"要不要调用"的依据，写清楚用途和示例
> - `input_schema` 用 JSON Schema 格式，Claude 靠这个生成合法的参数
> - 多个工具调用用 `Promise.all` 并发执行，比串行快 2-3 倍
> - 工具执行失败时，设置 `is_error: true`，Claude 会尝试其他方式回答

---

_本章完，下一章：MCP 协议详解 ➡️_
