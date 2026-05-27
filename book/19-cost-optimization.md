# 第19章：成本优化与 Token 管理

> "成本就像体重，不监控就会悄悄增长。" —— 老三

在实际生产环境中，API 成本往往是决定项目成败的关键因素之一。Claude Code SDK 虽然强大，但如果不加以控制和优化，成本可能会迅速攀升。本章将深入探讨如何监控、控制和优化 Claude Code SDK 的应用成本。

## 19.1 Token 与成本基础

### 19.1.1 Token 是什么？

Token 是 AI 模型处理文本的基本单位。在 Claude 模型中：

- **1 个 Token ≈ 4 个字符（英文）** 或 **≈ 1.5-2 个汉字（中文）**
- **1000 个 Token ≈ 750 个英文单词** 或 **≈ 500-600 个汉字**

Example:
```
英文: "Hello, how are you?" = 5 tokens
中文: "你好，最近怎么样？" = 10 tokens
```

### 19.1.2 Claude 模型的定价结构

Claude 模型按 **输入 Token** 和 **输出 Token** 分别计费：

| 模型 | 输入价格（每 M tokens） | 输出价格（每 M tokens） |
|------|------------------------|------------------------|
| Claude 3.5 Sonnet | $3.00 | $15.00 |
| Claude 3 Opus | $15.00 | $75.00 |
| Claude 3 Haiku | $0.25 | $1.25 |

> **重要发现：** 输出 Token 的价格通常是输入的 **5倍**！这意味着控制输出长度比控制输入长度更重要。

### 19.1.3 成本计算公式

```
总成本 = (输入 Token 数 × 输入单价) + (输出 Token 数 × 输出单价)
```

示例计算：
```javascript
// 一次 API 调用的成本计算
const inputTokens = 1000;
const outputTokens = 500;
const inputPricePerM = 3.00; // Claude 3.5 Sonnet
const outputPricePerM = 15.00;

const cost = (inputTokens / 1000000) * inputPricePerM 
           + (outputTokens / 1000000) * outputPricePerM;

console.log(`本次调用成本: $${cost.toFixed(6)}`);
// 输出: 本次调用成本: $0.0105
```

---

## 19.2 Token 计数与监控

### 19.2.1 使用 SDK 获取 Token 使用量

Claude Code SDK 在每次响应中都会返回 Token 使用信息：

```javascript
import { anthropic } from '@anthropic-ai/sdk';

const client = new anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

const response = await client.messages.create({
  model: 'claude-3-5-sonnet-20241022',
  max_tokens: 1024,
  messages: [{ role: 'user', content: '解释量子计算' }],
});

// 获取 Token 使用量
console.log('输入 Token:', response.usage.input_tokens);
console.log('输出 Token:', response.usage.output_tokens);
console.log('总成本:', calculateCost(response.usage));
```

### 19.2.2 构建 Token 计数器

为了持续监控成本，我们需要一个 Token 计数器：

```javascript
// token-counter.js
import { encoding_for_model } from 'tiktoken';

export class TokenCounter {
  constructor(model = 'claude-3-5-sonnet') {
    // 注意：tiktoken 是 OpenAI 的 tokenizer，Claude 使用不同的 tokenizer
    // 这里我们使用近似值，实际应使用 Anthropic 的 tokenizer
    this.encoder = encoding_for_model('gpt-4'); // 近似使用
  }

  /**
   * 计算文本的 Token 数（近似）
   */
  count(text) {
    if (!text) return 0;
    const tokens = this.encoder.encode(text);
    return tokens.length;
  }

  /**
   * 计算消息数组的 Token 数
   */
  countMessages(messages) {
    let total = 0;
    for (const msg of messages) {
      total += this.count(msg.content);
      total += 4; // 消息格式开销（近似）
    }
    return total;
  }
}

// 使用示例
const counter = new TokenCounter();
const messages = [
  { role: 'user', content: 'Hello' },
  { role: 'assistant', content: 'Hi there!' },
];

console.log('消息 Token 数:', counter.countMessages(messages));
```

**注意：** 上述代码使用 tiktoken 作为近似计算。Anthropic 提供了官方的 Token 计算工具，建议使用官方工具获得准确值。

### 19.2.3 精确 Token 计数（使用 Anthropic 官方方法）

Anthropic 提供了 `count_tokens` API 来精确计算 Token 数：

```javascript
// accurate-token-counter.js
import { Anthropic } from '@anthropic-ai/sdk';

export class AccurateTokenCounter {
  constructor(apiKey) {
    this.client = new Anthropic({ apiKey });
  }

  /**
   * 使用 Anthropic API 精确计算 Token 数
   */
  async count(text) {
    try {
      const response = await this.client.messages.countTokens({
        model: 'claude-3-5-sonnet-20241022',
        messages: [{ role: 'user', content: text }],
      });
      return response.input_tokens;
    } catch (error) {
      console.error('Token 计数失败:', error);
      // 降级到近似计算
      return Math.ceil(text.length / 4);
    }
  }
}

// 使用示例
const counter = new AccurateTokenCounter(process.env.ANTHROPIC_API_KEY);
const tokenCount = await counter.count('解释量子计算的基本原理');
console.log('精确 Token 数:', tokenCount);
```

---

## 19.3 成本监控仪表盘

### 19.3.1 设计思路

一个完整的成本监控系统应该包括：

1. **实时成本追踪** - 每次 API 调用的成本
2. **累计成本统计** - 按小时/天/月汇总
3. **预算预警** - 超过阈值时报警
4. **成本分解** - 按模型、用户、功能分解成本

### 19.3.2 实现成本追踪器

```javascript
// cost-tracker.js
import fs from 'fs/promises';
import path from 'path';

export class CostTracker {
  constructor(options = {}) {
    this.logFile = options.logFile || './cost-logs.jsonl';
    this.budget = options.budget || 100; // 月度预算（美元）
    this.alertThreshold = options.alertThreshold || 0.8; // 80% 预警
    this.pricing = {
      'claude-3-5-sonnet-20241022': { input: 3.00, output: 15.00 },
      'claude-3-opus-20240229': { input: 15.00, output: 75.00 },
      'claude-3-haiku-20240307': { input: 0.25, output: 1.25 },
    };
  }

  /**
   * 计算一次 API 调用的成本
   */
  calculateCost(model, inputTokens, outputTokens) {
    const price = this.pricing[model];
    if (!price) {
      console.warn(`未知模型: ${model}，使用默认价格`);
      return 0;
    }
    return (inputTokens / 1000000) * price.input 
         + (outputTokens / 1000000) * price.output;
  }

  /**
   * 记录一次 API 调用
   */
  async logCall(data) {
    const logEntry = {
      timestamp: new Date().toISOString(),
      model: data.model,
      inputTokens: data.inputTokens,
      outputTokens: data.outputTokens,
      cost: this.calculateCost(data.model, data.inputTokens, data.outputTokens),
      userId: data.userId || 'anonymous',
      feature: data.feature || 'unknown',
    };

    // 写入日志文件（JSONL 格式）
    await fs.appendFile(this.logFile, JSON.stringify(logEntry) + '\n');

    // 检查预算
    await this.checkBudget();

    return logEntry;
  }

  /**
   * 检查预算并预警
   */
  async checkBudget() {
    const monthlyCost = await this.getMonthlyCost();
    const percentage = monthlyCost / this.budget;

    if (percentage >= this.alertThreshold) {
      console.warn(`⚠️  预算预警: 本月已使用 ${(percentage * 100).toFixed(1)}% ($${monthlyCost.toFixed(2)}/$${this.budget})`);
    }

    if (percentage >= 1.0) {
      console.error(`🚨 预算超支: 本月已使用 $${monthlyCost.toFixed(2)}，超过预算 $${this.budget}`);
      throw new Error('预算超支');
    }
  }

  /**
   * 获取本月累计成本
   */
  async getMonthlyCost() {
    const logs = await this.loadLogs();
    const now = new Date();
    const thisMonth = `${now.getFullYear()}-${String(now.getMonth() + 1).padStart(2, '0')}`;

    return logs
      .filter(log => log.timestamp.startsWith(thisMonth))
      .reduce((sum, log) => sum + log.cost, 0);
  }

  /**
   * 加载所有日志
   */
  async loadLogs() {
    try {
      const content = await fs.readFile(this.logFile, 'utf-8');
      return content.trim().split('\n').map(line => JSON.parse(line));
    } catch (error) {
      return [];
    }
  }

  /**
   * 生成成本报告
   */
  async generateReport(period = 'daily') {
    const logs = await this.loadLogs();
    const now = new Date();

    // 按时间段过滤
    const filtered = logs.filter(log => {
      const logDate = new Date(log.timestamp);
      if (period === 'daily') {
        return logDate.toDateString() === now.toDateString();
      } else if (period === 'monthly') {
        return logDate.getMonth() === now.getMonth() 
            && logDate.getFullYear() === now.getFullYear();
      }
      return true;
    });

    // 按功能分解成本
    const byFeature = {};
    const byModel = {};
    let totalCost = 0;

    for (const log of filtered) {
      totalCost += log.cost;
      
      // 按功能
      byFeature[log.feature] = (byFeature[log.feature] || 0) + log.cost;
      
      // 按模型
      byModel[log.model] = (byModel[log.model] || 0) + log.cost;
    }

    return {
      period,
      totalCost,
      totalCalls: filtered.length,
      byFeature,
      byModel,
      avgCostPerCall: totalCost / filtered.length || 0,
    };
  }
}

// 使用示例
const tracker = new CostTracker({
  budget: 50, // 50美元/月
  alertThreshold: 0.7, // 70% 时预警
});

// 记录一次 API 调用
await tracker.logCall({
  model: 'claude-3-5-sonnet-20241022',
  inputTokens: 1000,
  outputTokens: 500,
  userId: 'user-123',
  feature: 'chat',
});

// 生成日报
const report = await tracker.generateReport('daily');
console.log('日报:', report);
```

### 19.3.3 可视化成本数据

```javascript
// cost-dashboard.js
import express from 'express';
import { CostTracker } from './cost-tracker.js';

const app = express();
const tracker = new CostTracker();

app.get('/cost/dashboard', async (req, res) => {
  const daily = await tracker.generateReport('daily');
  const monthly = await tracker.generateReport('monthly');
  const monthlyCost = await tracker.getMonthlyCost();

  res.json({
    currentMonth: {
      cost: monthlyCost,
      budget: tracker.budget,
      percentage: (monthlyCost / tracker.budget * 100).toFixed(1),
    },
    daily,
    monthly,
  });
});

app.listen(3000, () => {
  console.log('成本仪表盘运行在 http://localhost:3000/cost/dashboard');
});
```

---

## 19.4 Token 预算控制

### 19.4.1 为什么需要 Token 预算？

在生产环境中，我们需要：
- **防止意外超支** - 限制单次请求的最大成本
- **公平分配资源** - 为不同用户/功能分配不同的预算
- **优化性能** - 避免处理过长的输入

### 19.4.2 实现 Token 预算管理器

```javascript
// token-budget.js
export class TokenBudget {
  constructor(options = {}) {
    this.maxInputTokens = options.maxInputTokens || 10000;
    this.maxOutputTokens = options.maxOutputTokens || 4096;
    this.userBudgets = new Map(); // userId -> { daily, used }
    this.globalDailyBudget = options.globalDailyBudget || 1000000; // 全局每日预算
    this.globalUsedToday = 0;
  }

  /**
   * 检查是否允许处理该请求
   */
  checkRequest(userId, inputText) {
    const estimatedTokens = Math.ceil(inputText.length / 4); // 近似估算

    // 1. 检查输入长度
    if (estimatedTokens > this.maxInputTokens) {
      throw new Error(`输入过长: 估计 ${estimatedTokens} tokens，限制 ${this.maxInputTokens}`);
    }

    // 2. 检查用户预算
    const userBudget = this.getUserBudget(userId);
    if (userBudget.used + estimatedTokens > userBudget.daily) {
      throw new Error(`用户 ${userId} 的每日预算已用完`);
    }

    // 3. 检查全局预算
    if (this.globalUsedToday + estimatedTokens > this.globalDailyBudget) {
      throw new Error('全局每日预算已用完');
    }

    return true;
  }

  /**
   * 消耗 Token 预算
   */
  consume(userId, inputTokens, outputTokens) {
    // 更新用户预算
    const userBudget = this.getUserBudget(userId);
    userBudget.used += inputTokens + outputTokens;

    // 更新全局预算
    this.globalUsedToday += inputTokens + outputTokens;

    // 持久化（示例：写入 Redis）
    this.persist();
  }

  /**
   * 获取用户预算
   */
  getUserBudget(userId) {
    if (!this.userBudgets.has(userId)) {
      this.userBudgets.set(userId, {
        daily: 10000, // 默认每日 10K tokens
        used: 0,
      });
    }
    return this.userBudgets.get(userId);
  }

  /**
   * 设置用户预算
   */
  setUserBudget(userId, dailyBudget) {
    const budget = this.getUserBudget(userId);
    budget.daily = dailyBudget;
  }

  /**
   * 持久化预算数据
   */
  async persist() {
    // 这里可以写入数据库或 Redis
    // 示例：写入文件
    const data = {
      globalUsedToday: this.globalUsedToday,
      userBudgets: Array.from(this.userBudgets.entries()),
      date: new Date().toDateString(),
    };
    // await fs.writeFile('./budget.json', JSON.stringify(data));
  }

  /**
   * 重置每日预算（应该每天凌晨调用）
   */
  resetDaily() {
    this.globalUsedToday = 0;
    for (const [, budget] of this.userBudgets) {
      budget.used = 0;
    }
  }
}

// 使用示例
const budget = new TokenBudget({
  maxInputTokens: 5000,
  maxOutputTokens: 2048,
  globalDailyBudget: 500000,
});

// 在处理请求前检查
try {
  budget.checkRequest('user-123', '这是一个很长的输入...');
  
  // 执行 API 调用
  const response = await client.messages.create({...});
  
  // 消耗预算
  budget.consume('user-123', response.usage.input_tokens, response.usage.output_tokens);
} catch (error) {
  console.error('预算检查失败:', error.message);
}
```

---

## 19.5 Prompt 压缩技巧

### 19.5.1 为什么需要压缩 Prompt？

输入 Token 越多，成本越高。通过压缩 Prompt，我们可以：
- **降低成本** - 减少输入 Token 数
- **提高速度** - 模型处理更短的输入更快
- **提升质量** - 去除噪声，让模型聚焦核心内容

### 19.5.2 技巧1：去除冗余信息

❌ **不好的 Prompt：**
```
请分析以下代码，找出其中的 bug。代码是用 Python 写的。Python 是一种编程语言。
代码的功能是计算斐波那契数列。斐波那契数列是这样的：0, 1, 1, 2, 3, 5, 8...
现在请看代码：

def fib(n):
    if n <= 1:
        return n
    return fib(n-1) + fib(n-2)
```

✅ **优化后的 Prompt：**
```
分析以下 Python 代码的 bug：

def fib(n):
    if n <= 1:
        return n
    return fib(n-1) + fib(n-2)
```

**节省：** 约 80 tokens → 约 30 tokens（节省 62%）

### 19.5.3 技巧2：使用缩写和占位符

❌ **冗长：**
```
用户的问题是：如何实现登录功能？
请以专业的程序员身份回答这个问题，回答要详细、准确、易懂。
```

✅ **精简：**
```
Q: 如何实现登录功能？
A (专业、详细):
```

**节省：** 约 40 tokens → 约 15 tokens（节省 62%）

### 19.5.4 技巧3：Few-shot 示例优化

❌ **重复啰嗦的示例：**
```
示例1：
输入：2+2
输出：4
解释：这是加法

示例2：
输入：3+5
输出：8
解释：这是加法

现在请回答：
输入：4+6
```

✅ **精简示例：**
```
Q: 2+2 → A: 4
Q: 3+5 → A: 8
Q: 4+6 → A:
```

**节省：** 约 60 tokens → 约 20 tokens（节省 67%）

### 19.5.5 技巧4：使用系统消息缓存（Prompt Caching）

Claude 支持 Prompt Caching，可以缓存重复的系统消息：

```javascript
const response = await client.messages.create({
  model: 'claude-3-5-sonnet-20241022',
  system: [{
    type: 'text',
    text: '你是一个专业的 Python 程序员...', // 很长的系统提示
    cache_control: { type: 'ephemeral' }, // 启用缓存
  }],
  messages: [{ role: 'user', content: '如何实现登录？' }],
});

// 第二次调用时，系统消息会从缓存读取，成本大幅降低
const response2 = await client.messages.create({
  model: 'claude-3-5-sonnet-20241022',
  system: [{
    type: 'text',
    text: '你是一个专业的 Python 程序员...',
    cache_control: { type: 'ephemeral' },
  }],
  messages: [{ role: 'user', content: '如何实现注册？' }],
});
```

**成本对比：**
- 第一次调用：正常价格
- 后续调用：缓存读取价格（约 10% 的价格）

### 19.5.6 技巧5：动态截断历史消息

在多轮对话中，历史消息会不断增长。我们可以智能截断：

```javascript
function truncateMessages(messages, maxTokens = 10000) {
  let totalTokens = 0;
  const result = [];
  
  // 从最新的消息开始
  for (let i = messages.length - 1; i >= 0; i--) {
    const msg = messages[i];
    const tokens = estimateTokens(msg.content);
    
    if (totalTokens + tokens > maxTokens) {
      // 保留系统消息
      if (msg.role === 'system') {
        result.unshift(msg);
      }
      break;
    }
    
    result.unshift(msg);
    totalTokens += tokens;
  }
  
  return result;
}

// 使用
const truncated = truncateMessages(conversationHistory, 8000);
```

---

## 19.6 成本优化最佳实践

### 19.6.1 选择合适的模型

不同任务选择不同模型：

| 任务类型 | 推荐模型 | 原因 |
|---------|---------|------|
| 简单问答、分类 | Claude 3 Haiku | 成本低，速度快 |
| 代码生成、分析 | Claude 3.5 Sonnet | 性价比最高 |
| 复杂推理、创作 | Claude 3 Opus | 质量最高，但成本高 |

```javascript
function selectModel(task) {
  const modelMap = {
    'simple_qa': 'claude-3-haiku-20240307',
    'code_gen': 'claude-3-5-sonnet-20241022',
    'complex_reasoning': 'claude-3-opus-20240229',
  };
  return modelMap[task] || 'claude-3-5-sonnet-20241022';
}
```

### 19.6.2 设置合理的 max_tokens

避免让模型生成过长的内容：

```javascript
// ❌ 不设置 max_tokens，可能生成几千 tokens
const response1 = await client.messages.create({
  model: 'claude-3-5-sonnet',
  messages: [{ role: 'user', content: '解释量子计算' }],
});

// ✅ 设置合理的 max_tokens
const response2 = await client.messages.create({
  model: 'claude-3-5-sonnet',
  max_tokens: 500, // 限制输出长度
  messages: [{ role: 'user', content: '用 200 字解释量子计算' }],
});
```

### 19.6.3 使用流式响应 + 提前终止

如果用户已经得到所需信息，可以提前终止：

```javascript
import { Anthropic } from '@anthropic-ai/sdk';

const client = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

const stream = await client.messages.stream({
  model: 'claude-3-5-sonnet-20241022',
  max_tokens: 1024,
  messages: [{ role: 'user', content: '列出所有编程语言' }],
});

let totalLength = 0;
const maxLength = 500;

for await (const chunk of stream) {
  if (chunk.type === 'content_block_delta') {
    totalLength += chunk.delta.text.length;
    
    // 如果已经足够，提前终止
    if (totalLength > maxLength) {
      stream.abort(); // 终止流式响应
      console.log('提前终止，节省输出 Token');
      break;
    }
  }
}
```

### 19.6.4 批量处理请求

如果需要处理多个相似的请求，可以批量处理：

```javascript
// ❌ 逐个处理（成本高）
for (const question of questions) {
  await client.messages.create({
    model: 'claude-3-5-sonnet',
    messages: [{ role: 'user', content: question }],
  });
}

// ✅ 批量处理（成本低）
const batchPrompt = questions.map((q, i) => `${i + 1}. ${q}`).join('\n');
const response = await client.messages.create({
  model: 'claude-3-5-sonnet',
  messages: [{ role: 'user', content: `请依次回答以下问题：\n${batchPrompt}` }],
});
```

### 19.6.5 缓存常见回答

对于常见问题，可以缓存回答：

```javascript
// cache.js
import redis from 'redis';

const client = redis.createClient();

export async function cachedAsk(question) {
  // 检查缓存
  const cached = await client.get(question);
  if (cached) {
    console.log('缓存命中！');
    return cached;
  }
  
  // 调用 API
  const response = await anthropic.messages.create({
    model: 'claude-3-5-sonnet',
    messages: [{ role: 'user', content: question }],
  });
  
  const answer = response.content[0].text;
  
  // 缓存（1小时）
  await client.setEx(question, 3600, answer);
  
  return answer;
}
```

---

## 19.7 实战案例：构建成本监控系统

### 19.7.1 架构设计

```
┌─────────────┐
│  应用层     │
│  (API调用)  │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ 成本追踪层  │
│ (CostTracker)│
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ 预算控制层  │
│(TokenBudget) │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ 持久化层    │
│ (Redis/DB)  │
└─────────────┘
```

### 19.7.2 完整实现

```javascript
// cost-aware-client.js
import { Anthropic } from '@anthropic-ai/sdk';
import { CostTracker } from './cost-tracker.js';
import { TokenBudget } from './token-budget.js';

export class CostAwareClient {
  constructor(options = {}) {
    this.client = new Anthropic({ apiKey: options.apiKey });
    this.tracker = new CostTracker(options.costTracker);
    this.budget = new TokenBudget(options.tokenBudget);
  }

  async ask(userId, question, options = {}) {
    // 1. 预算检查
    this.budget.checkRequest(userId, question);

    // 2. 调用 API
    const response = await this.client.messages.create({
      model: options.model || 'claude-3-5-sonnet-20241022',
      max_tokens: options.maxTokens || 1024,
      messages: [{ role: 'user', content: question }],
    });

    // 3. 记录成本
    await this.tracker.logCall({
      model: response.model,
      inputTokens: response.usage.input_tokens,
      outputTokens: response.usage.output_tokens,
      userId,
      feature: options.feature || 'chat',
    });

    // 4. 消耗预算
    this.budget.consume(
      userId,
      response.usage.input_tokens,
      response.usage.output_tokens
    );

    return response.content[0].text;
  }
}

// 使用示例
const client = new CostAwareClient({
  apiKey: process.env.ANTHROPIC_API_KEY,
  costTracker: {
    budget: 100, // 100美元/月
    alertThreshold: 0.8,
  },
  tokenBudget: {
    maxInputTokens: 10000,
    maxOutputTokens: 4096,
    globalDailyBudget: 1000000,
  },
});

// 安全调用（自动追踪成本 + 控制预算）
try {
  const answer = await client.ask('user-123', '解释量子计算', {
    feature: 'education',
  });
  console.log(answer);
} catch (error) {
  console.error('请求失败:', error.message);
}
```

---

## 19.8 成本优化检查清单

在部署生产环境前，检查以下项目：

- [ ] **监控** - 是否实现了成本追踪？
- [ ] **预算** - 是否设置了月度/每日预算？
- [ ] **预警** - 预算达到 80% 时是否报警？
- [ ] **模型选择** - 是否根据任务选择了合适的模型？
- [ ] **max_tokens** - 是否设置了合理的输出长度限制？
- [ ] **Prompt 优化** - 是否去除了冗余信息？
- [ ] **缓存** - 是否使用了 Prompt Caching？
- [ ] **批量处理** - 相似请求是否批量处理？
- [ ] **常见回答缓存** - 是否缓存了常见问题的回答？
- [ ] **预算控制** - 是否为用户设置了 Token 预算？

---

## 19.9 小结

本章我们深入探讨了 Claude Code SDK 的成本优化与 Token 管理：

1. **Token 与成本基础** - 理解 Token 计数和定价结构
2. **成本监控** - 实现了成本追踪器和可视化仪表盘
3. **预算控制** - 构建了 Token 预算管理器
4. **Prompt 压缩** - 学习了 5 种压缩技巧
5. **最佳实践** - 总结了成本优化的实战经验

记住：**成本优化不是一次性的工作，而是持续的过程**。建议每周回顾成本数据，找出优化空间。

下一章，我们将学习**测试策略与 Mock 技巧**，确保我们的 SDK 应用质量可靠。

---

**练习题：**

1. 实现一个成本追踪器，记录每次 API 调用的成本并生成日报
2. 使用 Prompt Caching 优化一个多轮对话应用，对比优化前后的成本
3. 为一个团队聊天应用设计 Token 预算方案（不同用户角色有不同的预算）

---

*写于 2026-05-27 | 老三的 Claude Code SDK 编程指南*
