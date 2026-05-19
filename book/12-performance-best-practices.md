# 第12章：性能优化与最佳实践

> "快不是目的，稳才是。但能快又能稳，那才叫高手。" —— 老三

写一个能跑的 Claude Code SDK 程序不难，但写一个**快、稳、省**的程序，需要下一番功夫。这一章，我们从延迟优化、错误处理、日志监控三个维度，把你的 SDK 程序从「能用」提升到「能扛」。

---

## 12.1 减少延迟的技巧

延迟是用户体验的头号杀手。Claude API 的响应时间通常在 2-10 秒之间，但这不意味着你什么都做不了。

### 12.1.1 流式输出（Streaming）

最直接的优化——别等完整响应，边生成边输出。

```typescript
import { ClaudeCode } from '@anthropic-ai/claude-code';

const client = new ClaudeCode();

// ❌ 非流式：用户盯着空白等 5 秒
const result = await client.query('用 TypeScript 写一个快速排序');

// ✅ 流式：用户 0.3 秒就看到第一个字
const message = await client.query('用 TypeScript 写一个快速排序', {
  stream: true,
});

for await (const event of message) {
  if (event.type === 'content_block_delta') {
    process.stdout.write(event.delta.text ?? '');
  }
}
```

流式输出的核心价值不是减少总耗时，而是**减少用户感知的等待时间**。人类对"开始出现内容"的延迟极其敏感，对"内容持续输出"的容忍度高得多。

### 12.1.2 Prompt 缓存（Prompt Caching）

如果你反复发送相同的前缀（比如系统提示、工具定义、上下文文档），Prompt 缓存可以让 API 跳过重复处理，直接从缓存命中。

```typescript
// 在 system prompt 中标记缓存断点
const systemPrompt = `你是一个专业的代码审查助手。
以下是项目的编码规范：
- 使用 TypeScript strict 模式
- 函数不超过 20 行
- 必须有单元测试

<project_context>
${projectDocs}  // 这部分每次都一样，适合缓存
</project_context>`;

const result = await client.query('审查这段代码：' + code, {
  systemPrompt,
  // Anthropic API 支持 cache_control 标记
  // SDK 会自动处理缓存头
});
```

**缓存命中的效果：**
- 延迟降低 50%-80%
- 输入 token 费用降低 90%（缓存命中的 token 按 0.1x 计费）

**最佳实践：**
- 把稳定的上下文放在 prompt 前面
- 动态内容放后面
- 同一会话中复用 system prompt

### 12.1.3 并行 Tool 调用

当用户请求需要多个独立操作时，别串行等，并行跑。

```typescript
// ❌ 串行：查3个文件，等 3 × 2 = 6 秒
const file1 = await readFile('src/a.ts');
const file2 = await readFile('src/b.ts');
const file3 = await readFile('src/c.ts');

// ✅ 并行：3个文件同时读，等 2 秒
const [file1, file2, file3] = await Promise.all([
  readFile('src/a.ts'),
  readFile('src/b.ts'),
  readFile('src/c.ts'),
]);
```

在 Claude Code SDK 的工具定义中，如果多个工具调用之间没有依赖关系，Claude 会自动并行调用。你只需要确保工具定义清晰、无副作用冲突。

### 12.1.4 合理控制上下文长度

上下文越长，响应越慢。每多 1000 token，大约增加 50-100ms 延迟。

```typescript
// ❌ 把整个代码库塞进去
const allFiles = await glob('**/*.ts');
const allContent = await Promise.all(allFiles.map(f => readFile(f)));
const megaPrompt = allContent.join('\n');

// ✅ 只塞相关文件
const relevantFiles = await pickRelevantFiles(userQuery, allFiles);
const relevantContent = await Promise.all(
  relevantFiles.map(f => readFile(f))
);
const focusedPrompt = relevantContent.join('\n');
```

**精简策略：**
1. **摘要代替全文**：长文档先让 Claude 生成摘要，后续用摘要
2. **只发差异**：文件修改时只发 diff，不发全文
3. **分阶段处理**：先分析，再深入，别一次全给

---

## 12.2 错误处理与重试

网络请求天然不稳定。API 会限流、会超时、会偶尔 500。一个健壮的 SDK 程序必须优雅地处理这些情况。

### 12.2.1 常见错误类型

```typescript
import { ClaudeCode } from '@anthropic-ai/claude-code';

const client = new ClaudeCode();

try {
  const result = await client.query('你好');
} catch (error) {
  if (error.status === 429) {
    // 速率限制：请求太频繁
    console.log('被限流了，稍后重试');
  } else if (error.status === 500) {
    // 服务器错误：Anthropic 服务器出问题了
    console.log('服务器开小差了');
  } else if (error.status === 401) {
    // 认证失败：API Key 无效
    console.log('请检查 API Key');
  } else if (error.status === 400) {
    // 请求格式错误
    console.log('请求参数有问题：', error.message);
  } else if (error.code === 'ECONNRESET') {
    // 网络中断
    console.log('网络连接被重置');
  }
}
```

### 12.2.2 指数退避重试

重试不是无脑重试，要有策略。指数退避是最经典的方案：每次失败后等待时间翻倍。

```typescript
async function queryWithRetry(
  prompt: string,
  maxRetries = 3,
  baseDelay = 1000
): Promise<any> {
  const client = new ClaudeCode();
  
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await client.query(prompt);
    } catch (error: any) {
      // 不可重试的错误，直接抛出
      if (error.status === 401 || error.status === 400) {
        throw error;
      }
      
      // 可重试的错误
      if (attempt < maxRetries) {
        const delay = baseDelay * Math.pow(2, attempt);
        console.log(
          `请求失败 (${attempt + 1}/${maxRetries})，` +
          `${delay}ms 后重试...`
        );
        await sleep(delay);
      } else {
        throw new Error(
          `重试 ${maxRetries} 次后仍然失败：${error.message}`
        );
      }
    }
  }
}

function sleep(ms: number): Promise<void> {
  return new Promise(resolve => setTimeout(resolve, ms));
}

// 使用
const result = await queryWithRetry('分析这段代码', 3, 1000);
```

**重试原则：**
- `429`（限流）：必须重试，注意读 `Retry-After` 头
- `500`/`502`/`503`（服务器错误）：可重试
- `400`（参数错误）：不重试，修代码
- `401`/`403`（认证/权限）：不重试，修配置

### 12.2.3 超时控制

永远不要没有超时的网络请求。

```typescript
// 方式1：SDK 内置超时参数
const result = await client.query('生成代码', {
  timeout: 30000, // 30 秒超时
});

// 方式2：Promise.race 手动超时
async function queryWithTimeout(prompt: string, ms: number) {
  const client = new ClaudeCode();
  
  return Promise.race([
    client.query(prompt),
    new Promise((_, reject) =>
      setTimeout(
        () => reject(new Error(`请求超时 (${ms}ms)`)),
        ms
      )
    ),
  ]);
}

// 使用：30 秒超时
const result = await queryWithTimeout('写一个排序算法', 30000);
```

### 12.2.4 优雅降级

当主流程失败时，给用户一个可用的降级方案。

```typescript
async function smartQuery(prompt: string) {
  const client = new ClaudeCode();
  
  try {
    // 首选：使用最强模型
    return await client.query(prompt, {
      model: 'claude-sonnet-4-20250514',
    });
  } catch (error: any) {
    if (error.status === 429 || error.status >= 500) {
      console.log('主力模型不可用，切换到备用...');
      try {
        // 降级方案1：使用更轻量的模型
        return await client.query(prompt, {
          model: 'claude-haiku-3-20250307',
        });
      } catch {
        // 降级方案2：返回本地缓存结果
        const cached = await loadFromCache(prompt);
        if (cached) {
          console.log('使用缓存结果（可能不是最新的）');
          return cached;
        }
        // 最终降级：坦诚告知
        return {
          type: 'text',
          text: '抱歉，服务暂时不可用，请稍后再试。',
        };
      }
    }
    throw error;
  }
}
```

---

## 12.3 日志记录与监控

出了问题不可怕，可怕的是出了问题不知道为什么。

### 12.3.1 结构化日志

别用 `console.log` 一把梭，用结构化日志方便后续分析。

```typescript
import * as fs from 'fs';

interface LogEntry {
  timestamp: string;
  level: 'info' | 'warn' | 'error';
  event: string;
  duration?: number;
  tokens?: { input: number; output: number };
  error?: string;
  metadata?: Record<string, any>;
}

class SDKLogger {
  private logStream: fs.WriteStream;
  
  constructor(logFile = 'claude-sdk.log') {
    this.logStream = fs.createWriteStream(logFile, { flags: 'a' });
  }
  
  log(entry: Omit<LogEntry, 'timestamp'>) {
    const fullEntry: LogEntry = {
      ...entry,
      timestamp: new Date().toISOString(),
    };
    this.logStream.write(JSON.stringify(fullEntry) + '\n');
    
    // 错误级别也输出到控制台
    if (entry.level === 'error') {
      console.error(`[${fullEntry.timestamp}] ERROR: ${entry.error}`);
    }
  }
  
  close() {
    this.logStream.end();
  }
}

// 使用
const logger = new SDKLogger();

async function loggedQuery(prompt: string) {
  const client = new ClaudeCode();
  const startTime = Date.now();
  
  logger.log({ level: 'info', event: 'query_start', metadata: { prompt: prompt.slice(0, 100) } });
  
  try {
    const result = await client.query(prompt);
    const duration = Date.now() - startTime;
    
    logger.log({
      level: 'info',
      event: 'query_success',
      duration,
      tokens: {
        input: result.usage?.inputTokens ?? 0,
        output: result.usage?.outputTokens ?? 0,
      },
    });
    
    return result;
  } catch (error: any) {
    logger.log({
      level: 'error',
      event: 'query_error',
      duration: Date.now() - startTime,
      error: error.message,
    });
    throw error;
  }
}
```

### 12.3.2 关键指标监控

你需要关注的核心指标：

| 指标 | 含义 | 告警阈值 |
|------|------|----------|
| 请求延迟（P50/P95/P99） | API 响应速度 | P99 > 30s |
| 错误率 | 失败请求占比 | > 5% |
| Token 消耗 | 每次请求的 token 数 | 单次 > 100k |
| 缓存命中率 | Prompt 缓存命中比例 | < 30%（可优化） |
| 429 频率 | 被限流次数 | > 10次/小时 |

```typescript
// 简易指标收集器
class MetricsCollector {
  private latencies: number[] = [];
  private errors = 0;
  private total = 0;
  private tokensUsed = { input: 0, output: 0 };
  
  recordSuccess(duration: number, inputTokens: number, outputTokens: number) {
    this.latencies.push(duration);
    this.tokensUsed.input += inputTokens;
    this.tokensUsed.output += outputTokens;
    this.total++;
  }
  
  recordError() {
    this.errors++;
    this.total++;
  }
  
  getReport() {
    const sorted = [...this.latencies].sort((a, b) => a - b);
    return {
      total: this.total,
      errors: this.errors,
      errorRate: `${((this.errors / this.total) * 100).toFixed(1)}%`,
      p50: sorted[Math.floor(sorted.length * 0.5)] ?? 0,
      p95: sorted[Math.floor(sorted.length * 0.95)] ?? 0,
      p99: sorted[Math.floor(sorted.length * 0.99)] ?? 0,
      totalTokens: this.tokensUsed,
    };
  }
}

// 使用
const metrics = new MetricsCollector();

// 每小时输出报告
setInterval(() => {
  console.log('📊 性能报告：', JSON.stringify(metrics.getReport(), null, 2));
}, 3600_000);
```

### 12.3.3 成本控制

Token 就是钱。控制成本是生产环境的必修课。

```typescript
// 成本估算器（基于 Anthropic 公开定价）
function estimateCost(inputTokens: number, outputTokens: number, model: string) {
  const pricing: Record<string, { input: number; output: number }> = {
    'claude-sonnet-4-20250514': { input: 3, output: 15 },     // $/MTok
    'claude-haiku-3-20250307': { input: 0.8, output: 4 },     // $/MTok
  };
  
  const p = pricing[model] ?? pricing['claude-sonnet-4-20250514'];
  const cost = (inputTokens * p.input + outputTokens * p.output) / 1_000_000;
  return cost;
}

// 带预算控制的查询
class BudgetAwareClient {
  private totalCost = 0;
  private budgetLimit: number;
  private client: ClaudeCode;
  
  constructor(budgetLimit: number) {
    this.budgetLimit = budgetLimit;
    this.client = new ClaudeCode();
  }
  
  async query(prompt: string, model = 'claude-sonnet-4-20250514') {
    if (this.totalCost >= this.budgetLimit) {
      throw new Error(
        `预算已用完！已花费 $${this.totalCost.toFixed(4)}，` +
        `预算上限 $${this.budgetLimit}`
      );
    }
    
    const result = await this.client.query(prompt, { model });
    const cost = estimateCost(
      result.usage?.inputTokens ?? 0,
      result.usage?.outputTokens ?? 0,
      model
    );
    this.totalCost += cost;
    
    console.log(`本次花费: $${cost.toFixed(6)}，累计: $${this.totalCost.toFixed(4)}`);
    return result;
  }
  
  getRemainingBudget() {
    return this.budgetLimit - this.totalCost;
  }
}

// 使用：设置 $5 预算
const budgetClient = new BudgetAwareClient(5.0);
const result = await budgetClient.query('写一个 HTTP 服务器');
```

---

## 12.4 综合最佳实践清单

把上面的内容浓缩成一张清单，贴在你工位旁边：

### 🔧 延迟优化
- [ ] 启用流式输出（stream: true）
- [ ] 利用 Prompt 缓存，稳定内容放前面
- [ ] 无依赖的操作并行执行
- [ ] 控制上下文长度，只发相关内容
- [ ] 用轻量模型处理简单任务

### 🛡️ 错误处理
- [ ] 所有 API 调用都有 try-catch
- [ ] 实现指数退避重试
- [ ] 区分可重试和不可重试的错误
- [ ] 设置请求超时
- [ ] 有降级方案

### 📊 监控与成本
- [ ] 使用结构化日志
- [ ] 监控延迟 P50/P95/P99
- [ ] 跟踪错误率
- [ ] 记录 Token 消耗
- [ ] 设置预算上限

### 🧹 代码质量
- [ ] 系统提示与用户输入分离
- [ ] 工具定义简洁清晰
- [ ] 敏感信息不写死在代码里
- [ ] 优雅关闭（close client、flush 日志）

---

## 12.5 小结

这一章我们从三个维度优化了 SDK 程序：

1. **延迟**：流式输出、缓存、并行、精简上下文——让用户等得更少
2. **稳定性**：重试、超时、降级——让程序更抗揍
3. **可观测性**：结构化日志、指标监控、成本控制——让问题无处遁形

性能优化不是一次性的工作，而是一个持续迭代的过程。上线后多看日志，多看指标，哪里慢了优化哪里，哪里贵了精简哪里。

> "先让它跑起来，再让它跑得快，然后让它跑得稳。" —— 老三

下一章我们进入附录部分，提供完整的 API 参考手册。

---

_[下一章：附录A - API 参考手册 →](13-api-reference.md)_
