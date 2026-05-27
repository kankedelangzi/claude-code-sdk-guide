# 第15章：生产环境部署

> "Hello World 能跑是开始，能稳定跑才是本事。" —— 老三

从 Demo 到 Production，中间隔着十万八千里。你的 SDK 应用在本地跑得欢，一上线就翻车——API 调用爆预算了、响应延迟爆表了、错误日志一片红……这不是运气差，是没做好生产级部署。

这一章我们聊聊如何把 Claude Code SDK 应用稳稳地跑在生产环境：监控、成本、安全、高可用，一个都不能少。

---

## 15.1 监控与可观测性

生产环境最怕两件事：**不知道出问题了**、**出问题了不知道哪里出的问题**。监控体系要回答三个问题：

1. **系统健康吗？** — 可用性、响应时间、错误率
2. **成本可控吗？** — Token 消耗、API 调用次数
3. **用户满意吗？** — 业务指标、质量反馈

### 15.1.1 核心监控指标

```typescript
// monitoring/metrics.ts
import Anthropic from '@anthropic-ai/sdk';

interface RequestRecord {
  timestamp: number;
  latencyMs: number;
  success: boolean;
  inputTokens: number;
  outputTokens: number;
  model: string;
  errorMessage?: string;
}

interface Metrics {
  requestCount: number;
  successCount: number;
  errorCount: number;
  errorRate: number;
  avgLatencyMs: number;
  p50LatencyMs: number;
  p95LatencyMs: number;
  p99LatencyMs: number;
  inputTokens: number;
  outputTokens: number;
  totalTokens: number;
  estimatedCostUSD: number;
}

// 生产级指标收集器（集成 Prometheus / OpenTelemetry）
class MetricsCollector {
  private requests: RequestRecord[] = [];
  private readonly maxAgeMs = 3600 * 1000; // 保留最近1小时

  private readonly pricing: Record<string, { input: number; output: number }> = {
    'claude-opus-4-20250514':    { input: 15.0,  output: 75.0 },
    'claude-sonnet-4-20250514':   { input: 3.0,   output: 15.0 },
    'claude-haiku-4-20250514':    { input: 0.25,  output: 1.25 },
    'claude-3-5-haiku-20241022':  { input: 0.8,   output: 4.0 },
  };

  recordRequest(data: RequestRecord): void {
    this.requests.push(data);
    const cutoff = Date.now() - this.maxAgeMs;
    this.requests = this.requests.filter(r => r.timestamp > cutoff);
  }

  getMetrics(): Metrics {
    const total = this.requests.length;
    const success = this.requests.filter(r => r.success);
    const errors = this.requests.filter(r => !r.success);
    const latencies = success.map(r => r.latencyMs).sort((a, b) => a - b);
    const totalInput = this.requests.reduce((s, r) => s + r.inputTokens, 0);
    const totalOutput = this.requests.reduce((s, r) => s + r.outputTokens, 0);

    const percentile = (arr: number[], p: number): number => {
      if (arr.length === 0) return 0;
      return arr[Math.min(Math.floor(arr.length * p), arr.length - 1)];
    };

    let estimatedCostUSD = 0;
    for (const r of this.requests) {
      const p = this.pricing[r.model];
      if (p) {
        estimatedCostUSD += (r.inputTokens * p.input + r.outputTokens * p.output) / 1_000_000;
      }
    }

    return {
      requestCount: total,
      successCount: success.length,
      errorCount: errors.length,
      errorRate: total > 0 ? errors.length / total : 0,
      avgLatencyMs: latencies.length > 0 ? latencies.reduce((a, b) => a + b, 0) / latencies.length : 0,
      p50LatencyMs: percentile(latencies, 0.50),
      p95LatencyMs: percentile(latencies, 0.95),
      p99LatencyMs: percentile(latencies, 0.99),
      inputTokens: totalInput,
      outputTokens: totalOutput,
      totalTokens: totalInput + totalOutput,
      estimatedCostUSD: Math.round(estimatedCostUSD * 100) / 100,
    };
  }

  getErrorBreakdown(): Record<string, number> {
    const breakdown: Record<string, number> = {};
    for (const r of this.requests) {
      if (!r.success) {
        const key = r.errorMessage || 'unknown';
        breakdown[key] = (breakdown[key] || 0) + 1;
      }
    }
    return breakdown;
  }
}

// 使用示例：包装 SDK 调用，自动记录指标
const metrics = new MetricsCollector();

async function trackedChat(
  client: Anthropic,
  params: Anthropic.Messages.MessageCreateParams
) {
  const start = Date.now();
  try {
    const response = await client.messages.create(params);
    metrics.recordRequest({
      timestamp: Date.now(),
      latencyMs: Date.now() - start,
      success: true,
      inputTokens: response.usage.input_tokens,
      outputTokens: response.usage.output_tokens,
      model: params.model,
    });
    return response;
  } catch (err) {
    metrics.recordRequest({
      timestamp: Date.now(),
      latencyMs: Date.now() - start,
      success: false,
      inputTokens: 0,
      outputTokens: 0,
      model: params.model,
      errorMessage: (err as Error).message,
    });
    throw err;
  }
}
```

### 15.1.2 集成 Prometheus（生产推荐）

```typescript
// monitoring/prometheus.ts
import { register, Counter, Histogram, Gauge } from 'prom-client';
import express from 'express';

const sdkRequestCounter = new Counter({
  name: 'claude_sdk_requests_total',
  help: 'Total Claude SDK requests',
  labelNames: ['model', 'status'],
});

const sdkLatencyHistogram = new Histogram({
  name: 'claude_sdk_request_duration_seconds',
  help: 'SDK request latency in seconds',
  labelNames: ['model'],
  buckets: [0.1, 0.5, 1, 3, 5, 10, 30],
});

const sdkTokenCounter = new Counter({
  name: 'claude_sdk_tokens_total',
  help: 'Total tokens consumed by type',
  labelNames: ['model', 'type'],
});

const sdkCostGauge = new Gauge({
  name: 'claude_sdk_estimated_cost_usd_total',
  help: 'Estimated total cost in USD',
  labelNames: ['model'],
});

export function recordSDKMetrics(data: {
  model: string;
  success: boolean;
  latencyMs: number;
  inputTokens: number;
  outputTokens: number;
  costUSD: number;
}) {
  sdkRequestCounter.inc({ model: data.model, status: data.success ? 'success' : 'error' });
  sdkLatencyHistogram.observe({ model: data.model }, data.latencyMs / 1000);
  sdkTokenCounter.inc({ model: data.model, type: 'input' }, data.inputTokens);
  sdkTokenCounter.inc({ model: data.model, type: 'output' }, data.outputTokens);
  sdkCostGauge.inc({ model: data.model }, data.costUSD);
}

// 暴露 /metrics 端点
export function startMetricsServer(port: number = 9090): void {
  const app = express();
  app.get('/metrics', async (_req, res) => {
    res.set('Content-Type', register.contentType);
    res.end(await register.metrics());
  });
  app.listen(port, () => console.log(`Metrics server listening on :${port}/metrics`));
}
```

---

## 15.2 错误处理与重试策略

Anthropic API 是典型的分布式系统调用，网络抖动、限流、服务过载都可能发生。**没有重试机制的 SDK 应用，生产环境必死无疑。**

### 15.2.1 错误分类与重试判断

```typescript
import Anthropic from '@anthropic-ai/sdk';

type SDKError =
  | Anthropic.APIError
  | Anthropic.APIConnectionError
  | Anthropic.RateLimitError
  | Anthropic.InternalServerError
  | Anthropic.APITimeoutError
  | Anthropic.AuthenticationError
  | Anthropic.InvalidRequestError;

// 判断错误是否可重试（关键！）
function isRetryableError(err: unknown): boolean {
  if (err instanceof Anthropic.RateLimitError) return true;       // 429
  if (err instanceof Anthropic.InternalServerError) return true;   // 500
  if (err instanceof Anthropic.APIConnectionError) return true;     // 网络错误
  if (err instanceof Anthropic.APITimeoutError) return true;       // 超时
  // 兜底：状态码判断
  if (err instanceof Anthropic.APIError) {
    const status = err.status ?? 0;
    return status === 429 || status >= 500;
  }
  return false;
}

// 不可重试的错误（这些重试也没用）
function isFatalError(err: unknown): boolean {
  if (err instanceof Anthropic.AuthenticationError) return true;   // 401 API Key 错了
  if (err instanceof Anthropic.InvalidRequestError) return true;   // 400 参数错了
  if (err instanceof Anthropic.PermissionError) return true;        // 403 权限不足
  return false;
}
```

### 15.2.2 指数退避重试器

```typescript
// errors/retry.ts
async function withRetry<T>(
  fn: () => Promise<T>,
  options: {
    maxRetries?: number;
    baseDelayMs?: number;
    maxDelayMs?: number;
    jitter?: boolean;
    onRetry?: (attempt: number, err: Error, delayMs: number) => void;
  } = {}
): Promise<T> {
  const {
    maxRetries = 5,
    baseDelayMs = 1000,
    maxDelayMs = 30000,
    jitter = true,
    onRetry,
  } = options;

  let lastError: Error = new Error('Unknown error');

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (err) {
      lastError = err as Error;

      // 最后一次尝试或不可重试错误 → 直接抛出
      if (attempt === maxRetries || !isRetryableError(err)) {
        break;
      }

      // 指数退避 + 随机抖动（防止惊群效应）
      const delayMs = Math.min(
        baseDelayMs * Math.pow(2, attempt) + (jitter ? Math.random() * 500 : 0),
        maxDelayMs
      );

      onRetry?.(attempt + 1, lastError, delayMs);
      await sleep(delayMs);
    }
  }

  throw lastError;
}

function sleep(ms: number): Promise<void> {
  return new Promise(resolve => setTimeout(resolve, ms));
}
```

### 15.2.3 生产级封装：带重试 + 监控的 Client

```typescript
// sdk/production-client.ts
import Anthropic from '@anthropic-ai/sdk';
import { withRetry, isRetryableError } from './retry.js';
import { recordSDKMetrics } from '../monitoring/prometheus.js';

export class ProductionAnthropicClient {
  private client: Anthropic;

  constructor(apiKey: string) {
    this.client = new Anthropic({ apiKey });
  }

  async chat(
    params: Anthropic.Messages.MessageCreateParams,
    options?: { maxRetries?: number }
  ): Promise<Anthropic.Messages.Message> {
    return withRetry(
      async () => {
        const start = Date.now();
        const response = await this.client.messages.create(params);
        const latencyMs = Date.now() - start;

        const cost = this.estimateCost(params.model, response.usage);
        recordSDKMetrics({
          model: params.model,
          success: true,
          latencyMs,
          inputTokens: response.usage.input_tokens,
          outputTokens: response.usage.output_tokens,
          costUSD: cost,
        });

        return response;
      },
      {
        maxRetries: options?.maxRetries ?? 5,
        onRetry: (attempt, err) => {
          console.warn(`[Retry] #${attempt} failed: ${err.message}, retrying...`);
        },
      }
    );
  }

  async *chatStream(
    params: Anthropic.Messages.MessageCreateParams
  ): AsyncGenerator<Anthropic.Messages.MessageStreamEvent> {
    const stream = this.client.messages.stream(params);
    for await (const event of stream) {
      yield event;
    }
  }

  private estimateCost(
    model: string,
    usage: { input_tokens: number; output_tokens: number }
  ): number {
    const pricing: Record<string, { input: number; output: number }> = {
      'claude-opus-4-20250514':  { input: 15.0, output: 75.0 },
      'claude-sonnet-4-20250514':{ input: 3.0,  output: 15.0 },
      'claude-haiku-4-20250514': { input: 0.25, output: 1.25 },
    };
    const p = pricing[model];
    if (!p) return 0;
    return (usage.input_tokens * p.input + usage.output_tokens * p.output) / 1_000_000;
  }
}
```

---

## 15.3 成本控制

API 调用成本失控是生产环境最常见的事故来源。一个死循环或者没做长度限制的批量任务，分分钟烧掉几百美元。

### 15.3.1 Token 预算控制器

```typescript
// cost/budget.ts
interface CostBudget {
  dailyUSD: number;        // 每日预算上限
  monthlyUSD: number;      // 每月预算上限
  perRequestMaxUSD: number; // 单请求最大允许成本
}

class CostController {
  private dailySpend = 0;
  private monthlySpend = 0;
  private readonly budget: CostBudget;
  private readonly alertCallbacks: Array<(msg: string, spend: number) => void> = [];

  constructor(budget: CostBudget) {
    this.budget = budget;
  }

  onAlert(callback: (msg: string, spend: number) => void): void {
    this.alertCallbacks.push(callback);
  }

  // 请求前预检：估算成本，超预算直接拒绝
  checkBudgetBeforeRequest(
    model: string,
    estimatedInputTokens: number,
    estimatedOutputTokens: number
  ): { allowed: boolean; reason?: string } {
    const estimatedCost = this.estimateCost(model, estimatedInputTokens, estimatedOutputTokens);

    if (estimatedCost > this.budget.perRequestMaxUSD) {
      return {
        allowed: false,
        reason: `单请求预估成本 $${estimatedCost.toFixed(4)} 超过上限 $${this.budget.perRequestMaxUSD}`,
      };
    }

    if (this.dailySpend + estimatedCost > this.budget.dailyUSD) {
      return {
        allowed: false,
        reason: `今日预算已用尽（已花费 $${this.dailySpend.toFixed(2)} / 预算 $${this.budget.dailyUSD}）`,
      };
    }

    return { allowed: true };
  }

  // 记录实际花费
  recordSpend(model: string, usage: { input_tokens: number; output_tokens: number }): number {
    const cost = this.estimateCost(model, usage.input_tokens, usage.output_tokens);
    this.dailySpend += cost;
    this.monthlySpend += cost;

    // 80% 预算告警
    if (this.dailySpend > this.budget.dailyUSD * 0.8) {
      this.alertCallbacks.forEach(cb =>
        cb(
          `⚠️ 每日预算已使用 ${(this.dailySpend / this.budget.dailyUSD * 100).toFixed(0)}%：` +
          `$${this.dailySpend.toFixed(2)} / $${this.budget.dailyUSD}`,
          this.dailySpend
        )
      );
    }

    return cost;
  }

  getDailySpend(): number { return this.dailySpend; }
  getMonthlySpend(): number { return this.monthlySpend; }

  private estimateCost(model: string, inputTokens: number, outputTokens: number): number {
    const pricing: Record<string, { input: number; output: number }> = {
      'claude-opus-4-20250514':  { input: 15.0, output: 75.0 },
      'claude-sonnet-4-20250514':{ input: 3.0,  output: 15.0 },
      'claude-haiku-4-20250514': { input: 0.25, output: 1.25 },
    };
    const p = pricing[model];
    if (!p) return 0;
    return (inputTokens * p.input + outputTokens * p.output) / 1_000_000;
  }
}
```

### 15.3.2 集成到 SDK Client

```typescript
// sdk/production-client.ts（补充）
export class ProductionAnthropicClient {
  // ... 前面的代码 ...

  constructor(
    apiKey: string,
    private readonly costController?: CostController
  ) {
    this.client = new Anthropic({ apiKey });
  }

  async chat(
    params: Anthropic.Messages.MessageCreateParams,
    options?: { maxRetries?: number }
  ): Promise<Anthropic.Messages.Message> {
    // 预检预算（用 max_tokens 估算最大成本）
    if (this.costController) {
      const maxTokens = params.max_tokens ?? 4096;
      const inputTokens = this.estimateInputTokens(params.messages);
      const check = this.costController.checkBudgetBeforeRequest(
        params.model,
        inputTokens,
        maxTokens
      );
      if (!check.allowed) {
        throw new Error(`Budget exceeded: ${check.reason}`);
      }
    }

    const response = await withRetry(
      () => this.client.messages.create(params),
      { maxRetries: options?.maxRetries ?? 5 }
    );

    // 记录实际花费
    if (this.costController) {
      this.costController.recordSpend(params.model, response.usage);
    }

    return response;
  }

  private estimateInputTokens(messages: Anthropic.Messages.MessageParam[]): number {
    // 粗略估算：1 token ≈ 4 字符（英文）或 1.5 字符（中文）
    // 生产环境建议用官方 tokenizer，或直接用上一次响应的 usage.input_tokens
    return messages.reduce((sum, m) => {
      if (typeof m.content === 'string') return sum + Math.ceil(m.content.length / 4);
      return sum + 1000; // 多模态内容保守估算
    }, 0);
  }
}
```

---

## 15.4 安全：API Key 管理

**绝对不要把 API Key 写进代码、提交到 Git、或者硬编码在配置文件里。**

### 15.4.1 安全的 Key 管理方案

```typescript
// security/api-key.ts
import fs from 'fs';
import path from 'path';

// ✅ 推荐方案1：环境变量（最简单，适合中小项目）
// export ANTHROPIC_API_KEY="sk-ant-xxx" in .env

// ✅ 推荐方案2：密钥管理服务（AWS Secrets Manager / GCP Secret Manager / Vault）
// 适合多服务、多环境的生产系统

async function loadApiKey(): Promise<string> {
  // 1. 优先：环境变量
  const fromEnv = process.env.ANTHROPIC_API_KEY;
  if (fromEnv && fromEnv.startsWith('sk-ant-')) {
    return fromEnv;
  }

  // 2. 兜底：本地加密配置文件（仅开发环境）
  if (process.env.NODE_ENV === 'development') {
    const keyPath = path.join(process.cwd(), '.secrets', 'anthropic.key');
    if (fs.existsSync(keyPath)) {
      return fs.readFileSync(keyPath, 'utf-8').trim();
    }
  }

  throw new Error(
    'ANTHROPIC_API_KEY not found. Set environment variable or configure secret manager.'
  );
}

// ✅ 方案2示例：AWS Secrets Manager
// import { SecretsManagerClient, GetSecretValueCommand } from '@aws-sdk/client-secrets-manager';
//
// async function loadApiKeyFromAWS(secretName: string): Promise<string> {
//   const client = new SecretsManagerClient({ region: 'us-east-1' });
//   const cmd = new GetSecretValueCommand({ SecretId: secretName });
//   const result = await client.send(cmd);
//   const secret = JSON.parse(result.SecretString || '{}');
//   return secret.ANTHROPIC_API_KEY;
// }
```

### 15.4.2 .env 文件规范

```bash
# .env.example（提交到 Git，给同事参考）
ANTHROPIC_API_KEY=sk-ant-xxxxxxxxxxxxx
ANTHROPIC_BASE_URL=https://api.anthropic.com
ANTHROPIC_MODEL=claude-sonnet-4-20250514
COST_BUDGET_DAILY_USD=50
COST_BUDGET_MONTHLY_USD=500
METRICS_PORT=9090
LOG_LEVEL=info
```

```bash
# .gitignore（永远不要提交真实的 .env）
.env
.env.local
.secrets/
*.key
```

---

## 15.5 速率限制（Rate Limit）处理

Anthropic 对不同层级账户有速率限制（RPM：每分钟请求数，TPM：每分钟 Token 数）。超出限制会收到 `429 RateLimitError`。

### 15.5.1 速率限制类型与处理策略

| 限制类型 | 错误提示 | 处理策略 |
|---------|---------|---------|
| RPM 限制 | `rate_limit_error` (requests) | 降低并发、加队列、指数退避 |
| TPM 限制 | `rate_limit_error` (tokens) | 减小 batch 大小、拆分请求 |
| 并发限制 | `rate_limit_error` (concurrent) | 信号量控制并发数 |

### 15.5.2 Token Bucket 限速器（本地限流）

```typescript
// ratelimit/token-bucket.ts

// Token Bucket 算法：控制本地请求速率，避免触发 Anthropic 的 RPM 限制
class TokenBucket {
  private tokens: number;
  private