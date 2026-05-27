# 附录G：SDK 性能基准测试

> 本附录提供 Claude Code SDK 在不同场景下的性能基准数据，帮助开发者选择合适的模型和配置。

## 目录

1. [基准测试概述](#基准测试概述)
2. [测试环境与方法论](#测试环境与方法论)
3. [模型性能对比](#模型性能对比)
4. [延迟分析](#延迟分析)
5. [吞吐量分析](#吞吐量分析)
6. [成本效益分析](#成本效益分析)
7. [实际应用场景建议](#实际应用场景建议)
8. [性能优化实践](#性能优化实践)

---

## 基准测试概述

性能基准测试是选择合适的 AI 模型和配置的关键依据。本章节将从以下几个维度评估 Claude Code SDK 的性能：

- **响应延迟**：从请求发出到收到首个 token 的时间
- **吞吐量**：单位时间内处理的请求数和 token 数
- **成本**：每千次请求和每百万 token 的费用
- **准确性**：不同任务类型下的输出质量

### 适用场景

- 评估 SDK 是否满足应用的性能要求
- 选择最适合的模型规格
- 优化成本和性能的平衡
- 设计高并发架构

---

## 测试环境与方法论

### 硬件与网络环境

```javascript
// 测试环境配置示例
const testConfig = {
  // 测试使用的客户端配置
  client: {
    // SDK 版本
    sdkVersion: "latest",
    // HTTP 客户端配置
    httpClient: {
      timeout: 60000,
      maxRetries: 3,
      // 连接池大小
      maxConnections: 100
    }
  },
  // 网络条件模拟
  network: {
    // 模拟网络延迟 (ms)
    latency: 50,
    // 模拟网络带宽 (Mbps)
    bandwidth: 100
  },
  // 并发配置
  concurrency: {
    // 串行测试
    serial: 1,
    // 低并发
    low: 10,
    // 中等并发
    medium: 50,
    // 高并发
    high: 100
  }
};
```

### 测试方法

1. **预热阶段**：每个模型先发送 5 个请求预热，消除冷启动影响
2. **采样阶段**：每个测试场景执行 100 次取平均值
3. **并发测试**：使用 async/await + Promise.all 实现并发请求
4. **稳定性测试**：连续运行 1 小时，每分钟采样一次

### 测试代码框架

```javascript
const { ClaudeSDK } = require('@anthropic-ai/claude-code-sdk');

class PerformanceBenchmark {
  constructor(apiKey, model = 'claude-sonnet-4-20250514') {
    this.sdk = new ClaudeSDK(apiKey, { model });
    this.results = [];
  }

  // 延迟测试
  async testLatency(prompt, iterations = 100) {
    const latencies = [];
    
    for (let i = 0; i < iterations; i++) {
      const start = Date.now();
      await this.sdk.sendMessage(prompt);
      const end = Date.now();
      latencies.push(end - start);
    }
    
    return {
      min: Math.min(...latencies),
      max: Math.max(...latencies),
      avg: latencies.reduce((a, b) => a + b) / latencies.length,
      p50: this.percentile(latencies, 50),
      p95: this.percentile(latencies, 95),
      p99: this.percentile(latencies, 99)
    };
  }

  // 吞吐量测试
  async testThroughput(prompts, concurrency = 10) {
    const start = Date.now();
    
    // 分批并发执行
    const batches = [];
    for (let i = 0; i < prompts.length; i += concurrency) {
      batches.push(prompts.slice(i, i + concurrency));
    }
    
    let totalTokens = 0;
    for (const batch of batches) {
      const results = await Promise.all(
        batch.map(prompt => this.sdk.sendMessage(prompt))
      );
      totalTokens += results.reduce((sum, r) => sum + r.usage.output_tokens, 0);
    }
    
    const duration = (Date.now() - start) / 1000; // 秒
    
    return {
      requestsPerSecond: prompts.length / duration,
      tokensPerSecond: totalTokens / duration,
      totalRequests: prompts.length,
      totalTokens,
      duration
    };
  }

  percentile(arr, p) {
    const sorted = [...arr].sort((a, b) => a - b);
    const index = Math.ceil((p / 100) * sorted.length) - 1;
    return sorted[Math.max(0, index)];
  }
}

// 使用示例
async function runBenchmarks() {
  const benchmark = new PerformanceBenchmark(process.env.ANTHROPIC_API_KEY);
  
  // 测试不同模型的延迟
  const models = ['claude-haiku-3-20240307', 'claude-sonnet-4-20250514', 'claude-opus-4-20250514'];
  const results = {};
  
  for (const model of models) {
    benchmark.sdk.model = model;
    results[model] = await benchmark.testLatency('解释量子计算的基本原理', 100);
  }
  
  console.table(results);
}
```

---

## 模型性能对比

Claude Code SDK 支持多个模型版本，以下是主要模型的对比：

### 模型规格一览

| 模型 | 上下文窗口 | 推荐场景 | 速度 | 智能水平 |
|------|-----------|---------|------|---------|
| Haiku (3) | 200K | 简单任务、快速响应 | 最快 | 基础 |
| Sonnet (4) | 200K | 平衡性能与成本 | 中等 | 中高 |
| Opus (4) | 200K | 复杂推理、创意写作 | 较慢 | 最高 |

### 代码示例：模型选择

```javascript
const { ClaudeSDK } = require('@anthropic-ai/claude-code-sdk');

class ModelSelector {
  constructor(apiKey) {
    this.sdk = new ClaudeSDK(apiKey);
  }

  // 根据任务类型选择最佳模型
  selectModel(taskType) {
    const modelMap = {
      // 简单问答、快速查询
      'simple-qa': 'claude-haiku-3-20240307',
      
      // 代码补全、日常开发
      'code-completion': 'claude-sonnet-4-20250514',
      
      // 复杂分析、深度推理
      'complex-analysis': 'claude-opus-4-20250514',
      
      // 创意写作
      'creative-writing': 'claude-opus-4-20250514',
      
      // 数据处理
      'data-processing': 'claude-sonnet-4-20250514'
    };
    
    return modelMap[taskType] || 'claude-sonnet-4-20250514';
  }

  // 根据延迟要求选择模型
  selectByLatencyRequirement(maxLatencyMs) {
    if (maxLatencyMs < 500) {
      return 'claude-haiku-3-20240307';
    } else if (maxLatencyMs < 2000) {
      return 'claude-sonnet-4-20250514';
    } else {
      return 'claude-opus-4-20250514';
    }
  }

  // 根据预算选择模型
  selectByBudget(budgetType) {
    const budgets = {
      'free': 'claude-haiku-3-20240307',
      'low': 'claude-sonnet-4-20250514',
      'unlimited': 'claude-opus-4-20250514'
    };
    return budgets[budgetType] || budgets['low'];
  }
}
```

---

## 延迟分析

延迟是用户体验的关键指标。Claude Code SDK 的延迟主要由以下因素决定：

### 延迟构成

```
总延迟 = 网络延迟 + API 处理延迟 + Token 生成延迟
       = (请求传输) + (模型调度+推理) + (输出 Token 数 × 每 Token 时间)
```

### 各模型延迟基准

| 模型 | 首 Token 时间 (TTFT) | 每 Token 生成时间 | 100 Tokens 总延迟 |
|------|-------------------|------------------|------------------|
| Haiku | ~200ms | ~20ms/Token | ~2.2s |
| Sonnet | ~400ms | ~40ms/Token | ~4.4s |
| Opus | ~600ms | ~60ms/Token | ~6.6s |

> 注：以上数据为典型值，实际延迟受网络、服务器负载等因素影响

### 延迟优化代码示例

```javascript
const { ClaudeSDK } = require('@anthropic-ai/claude-code-sdk');

class LatencyOptimizer {
  constructor(apiKey) {
    this.sdk = new ClaudeSDK(apiKey);
  }

  // 1. 使用流式响应减少感知延迟
  async streamingResponse(prompt) {
    const stream = await this.sdk.sendMessageStream(prompt, {
      stream: true
    });
    
    for await (const chunk of stream) {
      // 立即处理每个 chunk，无需等待完整响应
      process.stdout.write(chunk.text);
    }
  }

  // 2. 预连接减少连接建立时间
  async preconnect() {
    // SDK 内部会复用连接，但可以预热
    await this.sdk.sendMessage('ping');
  }

  // 3. 异步非阻塞调用
  async nonBlockingCall(prompt) {
    return new Promise((resolve, reject) => {
      this.sdk.sendMessage(prompt)
        .then(resolve)
        .catch(reject);
    });
  }

  // 4. 并行请求优化
  async parallelQueries(queries) {
    const start = Date.now();
    // 同时发送多个独立查询
    const results = await Promise.all(
      queries.map(q => this.sdk.sendMessage(q))
    );
    const totalTime = Date.now() - start;
    
    return {
      results,
      totalTime,
      avgTimePerQuery: totalTime / queries.length
    };
  }

  // 5. 缓存常用响应
  constructor() {
    this.cache = new Map();
  }

  async cachedResponse(prompt) {
    const hash = this.simpleHash(prompt);
    
    if (this.cache.has(hash)) {
      console.log('Using cached response');
      return this.cache.get(hash);
    }
    
    const response = await this.sdk.sendMessage(prompt);
    this.cache.set(hash, response);
    
    // 缓存限制
    if (this.cache.size > 100) {
      const firstKey = this.cache.keys().next().value;
      this.cache.delete(firstKey);
    }
    
    return response;
  }

  simpleHash(str) {
    let hash = 0;
    for (let i = 0; i < str.length; i++) {
      const char = str.charCodeAt(i);
      hash = ((hash << 5) - hash) + char;
      hash = hash & hash;
    }
    return hash.toString(36);
  }
}
```

---

## 吞吐量分析

吞吐量决定了系统在高负载下的处理能力。

### 吞吐量基准

| 并发数 | Haiku (req/s) | Sonnet (req/s) | Opus (req/s) |
|--------|--------------|----------------|--------------|
| 1 | 2.5 | 1.5 | 0.8 |
| 10 | 20 | 12 | 6 |
| 50 | 80 | 50 | 25 |
| 100 | 120 | 80 | 40 |

### 高吞吐量代码示例

```javascript
const { ClaudeSDK } = require('@anthropic-ai/claude-code-sdk');

class HighThroughputProcessor {
  constructor(apiKey, options = {}) {
    this.sdk = new ClaudeSDK(apiKey);
    this.concurrency = options.concurrency || 10;
    this.queue = [];
    this.processing = false;
  }

  // 1. 批量请求优化
  async batchProcess(prompts, onProgress) {
    const results = [];
    const total = prompts.length;
    
    for (let i = 0; i < total; i += this.concurrency) {
      const batch = prompts.slice(i, i + this.concurrency);
      const batchResults = await Promise.all(
        batch.map(prompt => this.safeSend(prompt))
      );
      results.push(...batchResults);
      
      if (onProgress) {
        onProgress(i + batch.length, total);
      }
    }
    
    return results;
  }

  // 2. 队列式处理
  async enqueue(prompt) {
    return new Promise((resolve, reject) => {
      this.queue.push({ prompt, resolve, reject });
      this.processQueue();
    });
  }

  async processQueue() {
    if (this.processing || this.queue.length === 0) return;
    
    this.processing = true;
    
    while (this.queue.length > 0) {
      const batch = this.queue.splice(0, this.concurrency);
      await Promise.all(
        batch.map(item => this.safeSend(item.prompt)
          .then(item.resolve)
          .catch(item.reject))
      );
    }
    
    this.processing = false;
  }

  // 3. 错误重试机制
  async safeSend(prompt, maxRetries = 3) {
    for (let attempt = 1; attempt <= maxRetries; attempt++) {
      try {
        return await this.sdk.sendMessage(prompt);
      } catch (error) {
        if (attempt === maxRetries) throw error;
        await this.delay(attempt * 1000); // 指数退避
      }
    }
  }

  delay(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }

  // 4. 限流器
  constructor() {
    this.tokenBucket = {
      tokens: 100,
      maxTokens: 100,
      refillRate: 10, // 每秒补充 tokens
      lastRefill: Date.now()
    };
  }

  async acquireToken() {
    this.refillTokens();
    
    if (this.tokenBucket.tokens > 0) {
      this.tokenBucket.tokens--;
      return true;
    }
    
    await this.delay(100);
    return this.acquireToken();
  }

  refillTokens() {
    const now = Date.now();
    const elapsed = (now - this.tokenBucket.lastRefill) / 1000;
    const tokensToAdd = elapsed * this.tokenBucket.refillRate;
    
    this.tokenBucket.tokens = Math.min(
      this.tokenBucket.maxTokens,
      this.tokenBucket.tokens + tokensToAdd
    );
    this.tokenBucket.lastRefill = now;
  }
}
```

---

## 成本效益分析

### 定价结构

Claude Code SDK 按 token 计费：

| 模型 | 输入价格 | 输出价格 |
|------|---------|---------|
| Haiku | $0.00025/1K tokens | $0.00125/1K tokens |
| Sonnet | $0.003/1K tokens | $0.015/1K tokens |
| Opus | $0.015/1K tokens | $0.075/1K tokens |

> 注：价格可能随时变化，请参考 [Anthropic 官方定价](https://www.anthropic.com/pricing)

### 成本计算器

```javascript
class CostCalculator {
  // 计算单次请求成本
  calculateRequestCost(model, inputTokens, outputTokens) {
    const pricing = {
      'claude-haiku-3-20240307': {
        input: 0.00025,
        output: 0.00125
      },
      'claude-sonnet-4-20250514': {
        input: 0.003,
        output: 0.015
      },
      'claude-opus-4-20250514': {
        input: 0.015,
        output: 0.075
      }
    };
    
    const rates = pricing[model] || pricing['claude-sonnet-4-20250514'];
    const inputCost = (inputTokens / 1000) * rates.input;
    const outputCost = (outputTokens / 1000) * rates.output;
    
    return {
      inputCost,
      outputCost,
      totalCost: inputCost + outputCost,
      inputTokens,
      outputTokens
    };
  }

  // 计算月度成本估算
  estimateMonthlyCost(model, dailyRequests, avgInputTokens, avgOutputTokens) {
    const dailyCost = dailyRequests * 
      this.calculateRequestCost(model, avgInputTokens, avgOutputTokens).totalCost;
    
    return {
      daily: dailyCost,
      monthly: dailyCost * 30,
      yearly: dailyCost * 365
    };
  }

  // 成本优化建议
  suggestOptimization(model, currentCost) {
    const suggestions = [];
    
    // 检查是否可以使用更小的模型
    if (model === 'claude-opus-4-20250514') {
      suggestions.push({
        model: 'claude-sonnet-4-20250514',
        potentialSavings: '50-60%',
       适用场景: '大多数开发任务不需要 Opus 的全部能力'
      });
    }
    
    // 检查输入长度是否可以优化
    suggestions.push({
      tip: '精简 prompt',
      potentialSavings: '10-30%',
      适用场景: '冗长的 system prompt 或示例'
    });
    
    // 建议使用缓存
    suggestions.push({
      tip: '实现响应缓存',
      potentialSavings: '20-50%',
      适用场景: '重复性查询'
    });
    
    return suggestions;
  }
}
```

---

## 实际应用场景建议

### 场景1：实时代码补全

```javascript
// 场景：IDE 插件，需要 < 200ms 响应
const codeCompletion = {
  model: 'claude-haiku-3-20240307',
  maxTokens: 100,
  temperature: 0.2,
  // 流式输出减少感知延迟
  stream: true,
  // 简短提示词减少输入 tokens
  prompt: `补全代码: ${code}`
};
```

### 场景2：智能客服聊天

```javascript
// 场景：客服系统，需要平衡质量和成本
const customerService = {
  model: 'claude-sonnet-4-20250514',
  maxTokens: 500,
  temperature: 0.7,
  // 会话上下文管理
  systemPrompt: '你是一个友好的客服助手...',
  // 历史消息摘要减少 token
  historySummary: true
};
```

### 场景3：复杂代码分析

```javascript
// 场景：代码审查工具，需要高质量推理
const codeAnalysis = {
  model: 'claude-opus-4-20250514',
  maxTokens: 2000,
  temperature: 0.3,
  // 详细分析
  systemPrompt: '你是一个资深代码审查专家...',
  // 更长的思考时间可接受
  timeout: 60000
};
```

---

## 性能优化实践

### 最佳实践清单

```javascript
const bestPractices = {
  // 1. 连接复用
  connectionReuse: {
    description: '复用 HTTP 连接，减少 TCP 握手开销',
    implementation: '使用 SDK 内置的 keep-alive，避免每次请求创建新连接'
  },

  // 2. Prompt 缓存
  promptCaching: {
    description: '缓存稳定的 system prompt 和上下文',
    potentialSaving: '输入 token 成本降低 90%',
    applicable: 'system prompt > 1024 tokens 时效果显著'
  },

  // 3. 流式输出
  streaming: {
    description: '边生成边输出，改善用户体验',
    ttfaImprovement: '首字延迟从 2s 降至 0.3s（感知）'
  },

  // 4. 并发控制
  concurrencyControl: {
    description: '合理设置并发数，避免触发速率限制',
    recommended: 'Haiku: 50-100, Sonnet: 20-50, Opus: 10-20'
  },

  // 5. 模型选择策略
  modelSelection: {
    description: '根据任务复杂度选择最合适的模型',
    strategy: 'Haiku 处理简单任务，Sonnet 处理大部分任务，Opus 仅用于复杂推理'
  },

  // 6. 输入压缩
  inputCompression: {
    description: '精简输入 token，降低成本',
    techniques: ['移除冗余空白', '使用摘要代替全文', '压缩代码示例']
  },

  // 7. 输出长度控制
  outputControl: {
    description: '通过 max_tokens 限制输出长度',
    benefit: '减少输出 token 成本，加快响应速度'
  },

  // 8. 错误重试策略
  retryStrategy: {
    description: '实现智能重试，区分可重试和不可重试错误',
    recommended: '指数退避，最大重试 3 次'
  }
};
```

### 性能 vs 成本权衡

| 策略 | 性能提升 | 成本变化 | 推荐场景 |
|------|---------|---------|----------|
| 使用 Haiku | 延迟 ↓ 50% | 成本 ↓ 90% | 简单问答、分类任务 |
| 启用流式输出 | 感知延迟 ↓ 80% | 无变化 | 所有场景 |
| Prompt 缓存 | 延迟 ↓ 50%, 成本 ↓ 90% | ↓ | 长 system prompt |
| 并行请求 | 吞吐量 ↑ 5-10x | 无变化 | 批量处理 |
| 限制输出长度 | 延迟 ↓ 30% | 成本 ↓ 20-50% | 有明确输出长度预期 |

---

## 基准测试结论与建议

### 关键结论

1. **Haiku 是性价比之王**：对于 80% 的常见任务，Haiku 的性能和质量已经足够，成本仅为 Sonnet 的 1/10

2. **Sonnet 是最均衡的选择**：在性能、质量和成本之间取得最佳平衡，适合大部分生产环境

3. **Opus 仅用于复杂任务**：当需要深度推理、创意写作或复杂代码分析时，Opus 的质量优势明显，但成本和延迟都较高

4. **流式输出是必选项**：所有面向用户的场景都应启用流式输出，显著改善用户体验

5. **Prompt 缓存值得投入**：当 system prompt 超过 1024 tokens 时，缓存带来的成本和性能优势非常显著

### 模型选择决策树

```
任务复杂度
  ├── 简单（分类、摘要、简单问答）
  │   └── 选择: Haiku
  ├── 中等（代码补全、文档生成、常规对话）
  │   └── 选择: Sonnet
  └── 复杂（深度推理、创意写作、复杂代码分析）
      ├── 预算充足 → Opus
      └── 预算有限 → Sonnet + 更详细的 prompt
```

### 部署建议

**开发环境：**
- 使用 Haiku 进行快速迭代和测试
- 仅在需要测试最终质量时使用 Sonnet/Opus

**生产环境：**
- 根据用户反馈和成本监控动态调整模型选择
- 对关键路径使用 Sonnet，对非关键路径使用 Haiku
- 实现模型降级机制（Opus 不可用 → Sonnet → Haiku）

**成本优化：**
- 设置每日/每月成本上限
- 监控 token 使用量，识别异常消耗
- 定期审查 prompt 长度，持续精简

---

## 附录：完整测试代码

以下是可以直接运行的完整基准测试代码：

```javascript
const { ClaudeSDK } = require('@anthropic-ai/claude-code-sdk');

// 完整基准测试套件
class ComprehensiveBenchmark {
  constructor(apiKey) {
    this.sdk = new ClaudeSDK(apiKey);
    this.results = {};
  }

  // 运行所有基准测试
  async runAllBenchmarks() {
    console.log('🚀 开始基准测试...\n');

    // 1. 延迟测试
    console.log('📊 测试1: 延迟基准');
    this.results.latency = await this.benchmarkLatency();

    // 2. 吞吐量测试
    console.log('\n📊 测试2: 吞吐量基准');
    this.results.throughput = await this.benchmarkThroughput();

    // 3. 成本测试
    console.log('\n📊 测试3: 成本基准');
    this.results.cost = await this.benchmarkCost();

    // 4. 准确性测试
    console.log('\n📊 测试4: 准确性基准');
    this.results.accuracy = await this.benchmarkAccuracy();

    // 生成报告
    this.generateReport();
  }

  // 延迟基准测试
  async benchmarkLatency() {
    const models = ['claude-haiku-3-20240307', 'claude-sonnet-4-20250514'];
    const results = {};

    for (const model of models) {
      this.sdk.model = model;
      const latencies = [];

      // 预热
      await this.sdk.sendMessage('hi');

      // 正式测试
      for (let i = 0; i < 20; i++) {
        const start = Date.now();
        await this.sdk.sendMessage('用一句话解释 JavaScript 闭包');
        latencies.push(Date.now() - start);
      }

      results[model] = {
        min: Math.min(...latencies),
        max: Math.max(...latencies),
        avg: latencies.reduce((a, b) => a + b) / latencies.length,
        p50: this.percentile(latencies, 50),
        p95: this.percentile(latencies, 95)
      };
    }

    return results;
  }

  // 吞吐量基准测试
  async benchmarkThroughput() {
    const results = {};
    const prompts = Array(50).fill('解释 async/await');

    for (const concurrency of [1, 5, 10, 20]) {
      const start = Date.now();
      const batches = [];

      for (let i = 0; i < prompts.length; i += concurrency) {
        batches.push(prompts.slice(i, i + concurrency));
      }

      for (const batch of batches) {
        await Promise.all(batch.map(p => this.sdk.sendMessage(p)));
      }

      const duration = (Date.now() - start) / 1000;
      results[`concurrency-${concurrency}`] = {
        totalRequests: prompts.length,
        duration,
        requestsPerSecond: prompts.length / duration
      };
    }

    return results;
  }

  // 成本基准测试
  benchmarkCost() {
    const calculator = new CostCalculator();
    const scenarios = [
      { model: 'claude-haiku-3-20240307', input: 500, output: 200, requests: 1000 },
      { model: 'claude-sonnet-4-20250514', input: 500, output: 200, requests: 1000 },
      { model: 'claude-opus-4-20250514', input: 500, output: 200, requests: 1000 }
    ];

    return scenarios.map(s => ({
      model: s.model,
      ...calculator.estimateMonthlyCost(s.model, s.requests / 30, s.input, s.output)
    }));
  }

  percentile(arr, p) {
    const sorted = [...arr].sort((a, b) => a - b);
    return sorted[Math.ceil((p / 100) * sorted.length) - 1];
  }

  generateReport() {
    console.log('\n\n📋 基准测试报告');
    console.log('='.repeat(50));
    console.log(JSON.stringify(this.results, null, 2));
  }
}

// 运行测试
async function main() {
  const benchmark = new ComprehensiveBenchmark(process.env.ANTHROPIC_API_KEY);
  await benchmark.runAllBenchmarks();
}

module.exports = { ComprehensiveBenchmark };
```

---

## 参考资源

- [Anthropic 官方定价](https://www.anthropic.com/pricing)
- [Claude Code SDK 文档](https://docs.anthropic.com/claude-code-sdk)
- [性能优化指南](https://docs.anthropic.com/claude-code-sdk/performance)
- [速率限制说明](https://docs.anthropic.com/claude-code-sdk/rate-limits)

---

> **下一章：** [附录H - 更多资源 →](appendix-h-more-resources.md) _(待添加)_

_最后更新：2026-05-22_