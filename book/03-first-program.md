# 第3章：第一个 SDK 程序

> "千里之行，始于足下。第一个 SDK 程序，始于一个 `npm install`。" —— 老三

恭喜你完成了环境搭建！现在，让我们写下第一个 Claude Code SDK 程序。本章将带你从零开始，写出能跑的代码，理解 SDK 的基本工作流程。

---

## 3.1 Hello World：你的第一个 SDK 调用

### 3.1.1 创建项目

先创建一个新项目（如果你还没创建的话）：

```bash
# 创建项目目录
mkdir my-first-claude-app
cd my-first-claude-app

# 初始化 Node.js 项目
npm init -y

# 安装 Claude Code SDK
npm install @anthropic-ai/claude-code-sdk

# 安装 TypeScript（可选，但推荐）
npm install -D typescript @types/node
npx tsc --init
```

### 3.1.2 第一个程序：询问 Claude

创建 `src/hello.ts`：

```typescript
#!/usr/bin/env node
import { ClaudeCode } from '@anthropic-ai/claude-code-sdk';

async function main() {
  // 1. 初始化 Claude Code 客户端
  const claude = new ClaudeCode({
    apiKey: process.env.ANTHROPIC_API_KEY, // 从环境变量读取 API Key
  });

  // 2. 发送一个简单的消息
  const response = await claude.messages.create({
    model: 'claude-3-5-sonnet-20241022',
    maxTokens: 1024,
    messages: [
      {
        role: 'user',
        content: '用一句话介绍你自己，要幽默。',
      },
    ],
  });

  // 3. 输出响应
  console.log('Claude 说：');
  console.log(response.content[0].text);
}

main().catch(console.error);
```

运行它：

```bash
# 设置 API Key（如果你还没设置）
export ANTHROPIC_API_KEY="your-api-key-here"

# 运行程序
ts-node src/hello.ts
```

**预期输出：**

```
Claude 说：
我是一个由 Anthropic 训练的人工智能助手，虽然我没有身体，但我有一颗想要帮助你的心 —— 以及几乎无穷的知识库。
```

---

## 3.2 理解基本流程

让我们拆解一下刚才的程序，理解每一行代码在做什么。

### 3.2.1 初始化客户端

```typescript
const claude = new ClaudeCode({
  apiKey: process.env.ANTHROPIC_API_KEY,
});
```

- `ClaudeCode` 是 SDK 的入口类
- 你需要提供 `apiKey`（从 Anthropic Console 获取）
- 也可以使用 `ANTHROPIC_API_KEY` 环境变量，SDK 会自动读取

**进阶配置：**

```typescript
const claude = new ClaudeCode({
  apiKey: 'your-api-key',
  baseURL: 'https://api.anthropic.com', // 可选：自定义 API 端点
  timeout: 60000, // 可选：请求超时时间（毫秒）
  maxRetries: 3, // 可选：自动重试次数
});
```

### 3.2.2 发送消息

```typescript
const response = await claude.messages.create({
  model: 'claude-3-5-sonnet-20241022',
  maxTokens: 1024,
  messages: [
    {
      role: 'user',
      content: '用一句话介绍你自己，要幽默。',
    },
  ],
});
```

核心参数说明：

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `model` | `string` | ✅ | 模型名称，如 `claude-3-5-sonnet-20241022` |
| `maxTokens` | `number` | ✅ | 最大输出 Token 数 |
| `messages` | `Message[]` | ✅ | 对话消息数组 |
| `system` | `string` | ❌ | 系统提示词（可选） |
| `temperature` | `number` | ❌ | 温度参数，0-2，默认 1 |

### 3.2.3 处理响应

```typescript
console.log(response.content[0].text);
```

响应对象的结构：

```typescript
interface MessageResponse {
  id: string;               // 消息 ID
  type: 'message';          // 固定值
  role: 'assistant';        // 角色
  content: ContentBlock[];  // 内容块数组
  model: string;            // 使用的模型
  stopReason: 'end_turn' | 'max_tokens' | 'stop_sequence';
  usage: {
    inputTokens: number;
    outputTokens: number;
  };
}
```

**注意：** `content` 是一个数组，因为 Claude 可以返回多种类型的内容（文本、图片、工具调用等）。对于简单的文本响应，取 `content[0].text` 即可。

---

## 3.3 进阶示例：带系统提示的对话

让我们写一个更实用的例子：一个"Python 代码解释器"。

创建 `src/python-helper.ts`：

```typescript
import { ClaudeCode } from '@anthropic-ai/claude-code-sdk';

const claude = new ClaudeCode();

async function explainPythonCode(code: string) {
  const response = await claude.messages.create({
    model: 'claude-3-5-sonnet-20241022',
    maxTokens: 2048,
    system: `你是一个 Python 代码解释专家。
    
规则：
1. 用简洁的中文解释代码的功能
2. 指出潜在的 bug 或改进点
3. 如果代码有错误，给出修复建议`,
    messages: [
      {
        role: 'user',
        content: `请解释这段 Python 代码：\n\`\`\`python\n${code}\n\`\`\``,
      },
    ],
  });

  return response.content[0].text;
}

// 测试
const testCode = `
def fibonacci(n):
    if n <= 1:
        return n
    return fibonacci(n-1) + fibonacci(n-2)
`;

explainPythonCode(testCode).then((explanation) => {
  console.log('代码解释：\n');
  console.log(explanation);
  
  // 打印 Token 使用情况
  console.log('\n---');
  console.log('输入 Token：', '（需要解析 response.usage.inputTokens）');
  console.log('输出 Token：', '（需要解析 response.usage.outputTokens）');
});
```

**运行结果示例：**

```
代码解释：

这是一个计算斐波那契数列的递归函数。

【功能说明】
- 输入：一个整数 n
- 输出：第 n 个斐波那契数
- 逻辑：如果 n <= 1，返回 n；否则递归计算前两个数的和

【潜在问题】
1. 性能问题：递归实现的时间复杂度是 O(2^n)，n 稍大就会很慢
2. 没有输入验证：如果 n 是负数或浮点数，会无限递归
3. 没有缓存：重复计算了很多次相同的值

【改进建议】
建议使用动态规划或添加缓存（@lru_cache）来优化性能。

---
输入 Token：45
输出 Token：287
```

---

## 3.4 运行和调试

### 3.4.1 使用 ts-node 快速运行

```bash
# 安装 ts-node
npm install -D ts-node

# 运行 TypeScript 文件
npx ts-node src/hello.ts
```

### 3.4.2 使用 nodemon 实现热重载

```bash
# 安装 nodemon
npm install -D nodemon

# 在 package.json 中添加脚本
{
  "scripts": {
    "dev": "nodemon --watch src --exec ts-node src/hello.ts"
  }
}

# 运行开发模式
npm run dev
```

现在，每当你修改 `src/` 下的文件，程序会自动重启。

### 3.4.3 调试技巧

**1. 打印完整响应对象**

```typescript
console.log(JSON.stringify(response, null, 2));
```

**2. 处理错误**

```typescript
try {
  const response = await claude.messages.create({ /* ... */ });
  console.log(response.content[0].text);
} catch (error) {
  if (error.status === 401) {
    console.error('API Key 无效，请检查你的 ANTHROPIC_API_KEY');
  } else if (error.status === 429) {
    console.error('请求过于频繁，请稍后再试');
  } else {
    console.error('未知错误：', error);
  }
}
```

**3. 使用环境变量调试**

```bash
# 打印详细的 HTTP 请求日志
export DEBUG="anthropic:*"

# 运行程序
ts-node src/hello.ts
```

---

## 3.5 完整示例：命令行问答工具

让我们把学到的知识组合起来，写一个完整的命令行工具。

创建 `src/cli-chat.ts`：

```typescript
#!/usr/bin/env node
import { ClaudeCode } from '@anthropic-ai/claude-code-sdk';

const claude = new ClaudeCode();

async function chat(question: string) {
  console.log(`\n🙋 你：${question}\n`);
  console.log('🤔 Claude 正在思考...\n');

  try {
    const response = await claude.messages.create({
      model: 'claude-3-5-sonnet-20241022',
      maxTokens: 2048,
      messages: [
        {
          role: 'user',
          content: question,
        },
      ],
    });

    console.log(`🤖 Claude：${response.content[0].text}\n`);
    console.log(`📊 Token 使用：输入 ${response.usage.inputTokens}，输出 ${response.usage.outputTokens}`);
  } catch (error) {
    console.error('❌ 出错啦：', error.message);
  }
}

// 从命令行参数读取问题
const question = process.argv.slice(2).join(' ');

if (!question) {
  console.log('用法：npm run chat "你的问题"');
  console.log('示例：npm run chat "用 TypeScript 写一个快速排序"');
  process.exit(1);
}

chat(question);
```

在 `package.json` 中添加脚本：

```json
{
  "scripts": {
    "chat": "ts-node src/cli-chat.ts"
  }
}
```

运行：

```bash
npm run chat "用 TypeScript 写一个快速排序"
```

**输出示例：**

```
🙋 你：用 TypeScript 写一个快速排序

🤔 Claude 正在思考...

🤖 Claude：好的！以下是用 TypeScript 实现的快速排序：

\`\`\`typescript
function quickSort(arr: number[]): number[] {
  if (arr.length <= 1) return arr;
  
  const pivot = arr[Math.floor(arr.length / 2)];
  const left = arr.filter(x => x < pivot);
  const middle = arr.filter(x => x === pivot);
  const right = arr.filter(x => x > pivot);
  
  return [...quickSort(left), ...middle, ...quickSort(right)];
}

// 测试
console.log(quickSort([3, 6, 8, 10, 1, 2, 1]));
// 输出: [1, 1, 2, 3, 6, 8, 10]
\`\`\`

这个实现使用了函数式编程风格，易读但会创建多个中间数组...

📊 Token 使用：输入 15，输出 1024
```

---

## 3.6 本章小结

恭喜！你已经完成了第一个 Claude Code SDK 程序。让我们回顾一下学到的关键点：

1. **初始化客户端**：使用 `new ClaudeCode({ apiKey })` 创建客户端
2. **发送消息**：调用 `claude.messages.create()` 方法
3. **处理响应**：从 `response.content[0].text` 读取文本输出
4. **错误处理**：使用 `try-catch` 捕获 API 错误
5. **调试技巧**：打印完整响应、使用 `DEBUG` 环境变量

**下一步：**

在下一章中，我们将深入探讨 Claude Code SDK 的核心概念与架构，理解消息、内容块、工具调用等高级特性。准备好，我们要加速啦！🚀

---

## 练习题

1. **修改 Hello World 程序**：让用户输入问题，而不是硬编码在代码里（提示：使用 `readline` 模块）

2. **添加流式输出**：修改 `cli-chat.ts`，使用 `stream: true` 参数实现逐字输出效果（提示：查阅 SDK 文档的流式响应部分）

3. **挑战题**：写一个程序，读取文件 `input.txt` 的内容，发送给 Claude 进行总结，然后将结果保存到 `output.txt`

---

> **老三的Tips：**
> - 遇到 `401` 错误？检查你的 API Key 是否正确设置
> - 遇到 `429` 错误？你被限流了，加个重试逻辑，或者等一会儿再试
> - 想要更好的类型提示？使用 TypeScript，并导入 SDK 的类型定义：`import type { MessageResponse } from '@anthropic-ai/claude-code-sdk'`

---

_本章完，下一章：核心概念与架构 ➡️_
