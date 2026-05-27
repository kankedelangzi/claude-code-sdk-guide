# 第21章：错误处理与重试机制

> "一次失败叫 Bug，反复失败叫架构缺陷。" —— 老三

第12章我们聊过基础的错误处理和指数退避。那只是热身。这一章，我们要把错误处理从「能 catch 住」升级到「能自愈」——构建一个**生产级**的错误处理与重试系统。

你将学到：
- 错误的**精确分类**与分级处理策略
- 三种重试模式的实现与适用场景
- 断路器模式防止级联故障
- 错误聚合与降级策略
- 完整的 `ResilientClient` 实现

---

## 21.1 错误分类：不是所有错误都值得重试

SDK 调用出错了，你的第一反应是什么？重试！但等等——**有些错误重试只会浪费 Token 和时间**。

### 21.1.1 错误三分类

把 Anthropic API 可能抛出的错误分成三类：

| 分类 | 状态码 | 特征 | 是否重试 |
|------|--------|------|----------|
| **客户端错误** | 400, 401, 403, 404 | 请求本身有问题 | ❌ 修复请求 |
| **服务端错误** | 500, 502, 503 | Anthropic 侧故障 | ✅ 可重试 |
| **限流错误** | 429 | 请求太频繁 | ✅ 可重试（等一会） |
| **网络错误** | ECONNRESET, ETIMEDOUT | 网络抖动 | ✅ 可重试 |
| **业务错误** | - | 模型输出不符合预期 | ❌ 调整 Prompt |

关键洞察：**只有「暂时性错误」才值得重试**。永久性错误重试一万次还是错。

### 21.1.2 错误分类器实现

```typescript
// error-classifier.ts — 精确的错误分类器

/**
 * 错误严重级别
 */
export enum ErrorSeverity {
  /** 可忽略：不影响核心流程 */
  LOW = 'low',
  /** 需关注：可能影响部分功能 */
  MEDIUM = 'medium',
  /** 严重：核心流程受阻 */
  HIGH = 'high',
  /** 致命：系统不可用 */
  CRITICAL = 'critical',
}

/**
 * 错误可恢复性
 */
export enum ErrorRecoverability {
  /** 不可恢复：重试无意义，需要人工干预 */
  PERMANENT = 'permanent',
  /** 可恢复：重试可能成功 */
  TRANSIENT = 'transient',
  /** 有条件可恢复：需要特定策略（如降级） */
  CONDITIONAL = 'conditional',
}

/**
 * 分类后的错误信息
 */
export interface ClassifiedError {
  /** 原始错误 */
  original: Error;
  /** 错误分类 */
  category: ErrorCategory;
  /** 严重级别 */
  severity: ErrorSeverity;
  /** 可恢复性 */
  recoverability: ErrorRecoverability;
  /** 建议的重试延迟（毫秒），不可恢复时为 -1 */
  suggestedRetryDelay: number;
  /** 人类可读的错误描述 */
  description: string;
  /** 是否需要告警 */
  needsAlert: boolean;
}

export enum ErrorCategory {
  AUTH = 'auth',
  RATE_LIMIT = 'rate_limit',
  SERVER = 'server',
  NETWORK = 'network',
  VALIDATION = 'validation',
  CONTEXT_LENGTH = 'context_length',
  TOOL_ERROR = 'tool_error',
  OVERLOAD = 'overload',
  UNKNOWN = 'unknown',
}

/**
 * 错误分类器 — 核心工具
 */
export class ErrorClassifier {
  /**
   * 对错误进行分类
   */
  classify(error: unknown): ClassifiedError {
    // Anthropic SDK 错误格式
    const status = (error as any)?.status ?? (error as any)?.statusCode;
    const message = (error as any)?.message ?? String(error);
    const code = (error as any)?.code;

    // 1. 认证错误 — 401/403
    if (status === 401 || status === 403) {
      return {
        original: error as Error,
        category: ErrorCategory.AUTH,
        severity: ErrorSeverity.CRITICAL,
        recoverability: ErrorRecoverability.PERMANENT,
        suggestedRetryDelay: -1,
        description: `认证失败: ${message}。请检查 API Key 是否正确。`,
        needsAlert: true,
      };
    }

    // 2. 限流错误 — 429
    if (status === 429) {
      const retryAfter = this.parseRetryAfter(error);
      return {
        original: error as Error,
        category: ErrorCategory.RATE_LIMIT,
        severity: ErrorSeverity.MEDIUM,
        recoverability: ErrorRecoverability.TRANSIENT,
        suggestedRetryDelay: retryAfter ?? 5000,
        description: `API 限流: ${message}。建议等待 ${retryAfter ?? 5000}ms 后重试。`,
        needsAlert: false,
      };
    }

    // 3. 上下文过长 — 400 + prompt_is_too_long
    if (status === 400 && message.includes('too long')) {
      return {
        original: error as Error,
        category: ErrorCategory.CONTEXT_LENGTH,
        severity: ErrorSeverity.HIGH,
        recoverability: ErrorRecoverability.CONDITIONAL,
        suggestedRetryDelay: -1,
        description: `上下文超长: ${message}。需要截断或压缩 Prompt。`,
        needsAlert: false,
      };
    }

    // 4. 参数验证错误 — 400（非上下文过长）
    if (status === 400) {
      return {
        original: error as Error,
        category: ErrorCategory.VALIDATION,
        severity: ErrorSeverity.HIGH,
        recoverability: ErrorRecoverability.PERMANENT,
        suggestedRetryDelay: -1,
        description: `请求参数错误: ${message}`,
        needsAlert: false,
      };
    }

    // 5. 服务端过载 — 529
    if (status === 529) {
      return {
        original: error as Error,
        category: ErrorCategory.OVERLOAD,
        severity: ErrorSeverity.MEDIUM,
        recoverability: ErrorRecoverability.TRANSIENT,
        suggestedRetryDelay: 10000,
        description: `API 过载: ${message}。Anthropic 服务器繁忙，建议稍后重试。`,
        needsAlert: true,
      };
    }

    // 6. 服务端错误 — 500/502/503
    if (status >= 500 && status < 600) {
      return {
        original: error as Error,
        category: ErrorCategory.SERVER,
        severity: ErrorSeverity.MEDIUM,
        recoverability: ErrorRecoverability.TRANSIENT,
        suggestedRetryDelay: 2000,
        description: `服务端错误 (${status}): ${message}`,
        needsAlert: status === 500, // 500 可能是 bug，需要告警
      };
    }

    // 7. 网络错误
    if (code === 'ECONNRESET' || code === 'ETIMEDOUT' || code === 'ENOTFOUND' || code === 'EAI_AGAIN') {
      return {
        original: error as Error,
        category: ErrorCategory.NETWORK,
        severity: ErrorSeverity.MEDIUM,
        recoverability: ErrorRecoverability.TRANSIENT,
        suggestedRetryDelay: 1000,
        description: `网络错误 (${code}): ${message}`,
        needsAlert: code === 'ENOTFOUND',
      };
    }

    // 8. Tool 执行错误
    if (message.includes('tool_use') || message.includes('tool_result')) {
      return {
        original: error as Error,
        category: ErrorCategory.TOOL_ERROR,
        severity: ErrorSeverity.MEDIUM,
        recoverability: ErrorRecoverability.CONDITIONAL,
        suggestedRetryDelay: 0,
        description: `Tool 执行错误: ${message}`,
        needsAlert: false,
      };
    }

    // 9. 未知错误
    return {
      original: error as Error,
      category: ErrorCategory.UNKNOWN,
      severity: ErrorSeverity.HIGH,
      recoverability: ErrorRecoverability.CONDITIONAL,
      suggestedRetryDelay: 1000,
      description: `未知错误: ${message}`,
      needsAlert: true,
    };
  }

  /**
   * 解析 Retry-After 头信息
   */
  private parseRetryAfter(error: any): number | null {
    const headers = error?.headers;
    if (!headers) return null;

    const retryAfter = headers['retry-after'];
    if (!retryAfter) return null;

    // 可能是秒数或 HTTP 日期
    const seconds = Number(retryAfter);
    if (!isNaN(seconds)) {
      return seconds * 1000;
    }

    // 尝试解析为日期
    const date = new Date(retryAfter);
    if (!isNaN(date.getTime())) {
      return Math.max(0, date.getTime() - Date.now());
    }

    return null;
  }
}
```

使用示例：

```typescript
import { ErrorClassifier } from './error-classifier';

const classifier = new ErrorClassifier();

// 模拟一个 429 错误
const rateLimitError = {
  status: 429,
  message: 'Rate limit exceeded',
  headers: { 'retry-after': '30' },
};

const classified = classifier.classify(rateLimitError);
console.log(classified);
// {
//   category: 'rate_limit',
//   severity: 'medium',
//   recoverability: 'transient',
//   suggestedRetryDelay: 30000,
//   description: 'API 限流: Rate limit exceeded。建议等待 30000ms 后重试。',
//   needsAlert: false
// }

// 根据分类决定策略
if (classified.recoverability === 'permanent') {
  console.error('❌ 不可恢复，需要修复:', classified.description);
} else if (classified.recoverability === 'transient') {
  console.log(`⏳ 暂时性错误，${classified.suggestedRetryDelay}ms 后重试`);
} else {
  console.log('⚠️ 有条件可恢复，需要降级策略');
}
```

---

## 21.2 重试策略：三种模式详解

第12章讲了指数退避，这里我们系统化地介绍三种重试模式。

### 21.2.1 固定间隔重试

最简单，适合**限流错误**（429），因为服务器告诉你要等多久。

```typescript
// retry-fixed.ts — 固定间隔重试

interface FixedRetryOptions {
  /** 最大重试次数 */
  maxRetries: number;
  /** 固定等待时间（毫秒） */
  delayMs: number;
  /** 只对特定错误重试 */
  retryOn?: (error: unknown) => boolean;
}

async function retryWithFixedDelay<T>(
  fn: () => Promise<T>,
  options: FixedRetryOptions
): Promise<T> {
  const { maxRetries, delayMs, retryOn } = options;

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      // 最后一轮，不再重试
      if (attempt === maxRetries) {
        throw error;
      }

      // 自定义过滤
      if (retryOn && !retryOn(error)) {
        throw error;
      }

      console.log(
        `⏳ 第 ${attempt + 1}/${maxRetries} 次重试，等待 ${delayMs}ms...`
      );
      await sleep(delayMs);
    }
  }

  throw new Error('unreachable');
}

function sleep(ms: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

// 使用示例：处理限流
const result = await retryWithFixedDelay(
  () => client.messages.create({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 1024,
    messages: [{ role: 'user', content: 'Hello' }],
  }),
  {
    maxRetries: 3,
    delayMs: 5000, // 固定等 5 秒
    retryOn: (err) => (err as any).status === 429, // 只重试限流
  }
);
```

### 21.2.2 指数退避重试

最常用，适合**服务端错误和网络抖动**。每次失败等待时间指数增长。

```typescript
// retry-exponential.ts — 指数退避 + 抖动

interface ExponentialRetryOptions {
  /** 最大重试次数 */
  maxRetries: number;
  /** 初始延迟（毫秒） */
  baseDelayMs: number;
  /** 最大延迟（毫秒） */
  maxDelayMs: number;
  /** 退避倍数，默认 2 */
  backoffFactor?: number;
  /** 是否添加随机抖动 */
  jitter?: boolean;
  /** 可重试的错误判断 */
  retryable?: (error: unknown) => boolean;
}

async function retryWithExponentialBackoff<T>(
  fn: () => Promise<T>,
  options: ExponentialRetryOptions
): Promise<T> {
  const {
    maxRetries,
    baseDelayMs,
    maxDelayMs,
    backoffFactor = 2,
    jitter = true,
    retryable,
  } = options;

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (attempt === maxRetries) throw error;
      if (retryable && !retryable(error)) throw error;

      // 计算退避时间
      let delay = baseDelayMs * Math.pow(backoffFactor, attempt);

      // 添加抖动，防止"惊群效应"
      if (jitter) {
        delay = delay * (0.5 + Math.random() * 0.5);
      }

      // 不超过最大延迟
      delay = Math.min(delay, maxDelayMs);

      console.log(
        `⏳ 第 ${attempt + 1}/${maxRetries} 次重试，等待 ${Math.round(delay)}ms...`
      );
      await sleep(delay);
    }
  }

  throw new Error('unreachable');
}

// 使用示例
const result = await retryWithExponentialBackoff(
  () => client.messages.create({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 1024,
    messages: [{ role: 'user', content: '写一首诗' }],
  }),
  {
    maxRetries: 5,
    baseDelayMs: 1000,   // 首次等 1 秒
    maxDelayMs: 60000,   // 最多等 60 秒
    backoffFactor: 2,    // 1s → 2s → 4s → 8s → 16s
    jitter: true,        // 加随机抖动
    retryable: (err) => {
      const status = (err as any).status;
      return status >= 500 || status === 429 || status === undefined;
    },
  }
);
```

**为什么需要抖动（Jitter）？**

想象一个场景：100 个客户端同时被 429 限流，它们都在等 2 秒。2 秒后，100 个请求同时打过来——又限流了。加抖动后，这 100 个客户端会在 1~2 秒之间随机分散重试，避免了「惊群效应」。

### 21.2.3 自适应重试

最智能，根据**历史成功率**动态调整重试策略。

```typescript
// retry-adaptive.ts — 自适应重试

interface RetryStats {
  /** 总尝试次数 */
  attempts: number;
  /** 成功次数 */
  successes: number;
  /** 连续失败次数 */
  consecutiveFailures: number;
  /** 最近 N 次的错误时间戳 */
  recentErrors: number[];
}

interface AdaptiveRetryOptions {
  /** 最大重试次数 */
  maxRetries: number;
  /** 基础延迟 */
  baseDelayMs: number;
  /** 最大延迟 */
  maxDelayMs: number;
  /** 窗口大小（统计最近多少秒的错误） */
  errorWindowSeconds?: number;
  /** 失败率阈值（超过此值认为服务不健康） */
  failureRateThreshold?: number;
}

class AdaptiveRetry {
  private stats: RetryStats = {
    attempts: 0,
    successes: 0,
    consecutiveFailures: 0,
    recentErrors: [],
  };

  constructor(private options: AdaptiveRetryOptions) {}

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    const { maxRetries, baseDelayMs, maxDelayMs } = this.options;
    const windowSeconds = this.options.errorWindowSeconds ?? 60;
    const threshold = this.options.failureRateThreshold ?? 0.5;

    for (let attempt = 0; attempt <= maxRetries; attempt++) {
      try {
        const result = await fn();
        this.recordSuccess();
        return result;
      } catch (error) {
        this.recordFailure();

        if (attempt === maxRetries) throw error;

        // 根据错误率动态调整延迟
        const failureRate = this.getRecentFailureRate(windowSeconds);
        let delay: number;

        if (failureRate > threshold || this.stats.consecutiveFailures > 3) {
          // 错误率过高，使用更长延迟
          delay = Math.min(baseDelayMs * Math.pow(3, attempt), maxDelayMs);
          console.log(
            `🔴 服务疑似不健康（失败率 ${(failureRate * 100).toFixed(1)}%，` +
            `连续失败 ${this.stats.consecutiveFailures} 次），` +
            `使用保守延迟 ${Math.round(delay)}ms`
          );
        } else {
          // 正常退避
          delay = Math.min(baseDelayMs * Math.pow(2, attempt), maxDelayMs);
        }

        // 加抖动
        delay *= 0.5 + Math.random() * 0.5;
        await sleep(delay);
      }
    }

    throw new Error('unreachable');
  }

  private recordSuccess(): void {
    this.stats.attempts++;
    this.stats.successes++;
    this.stats.consecutiveFailures = 0;
  }

  private recordFailure(): void {
    this.stats.attempts++;
    this.stats.consecutiveFailures++;
    this.stats.recentErrors.push(Date.now());
  }

  private getRecentFailureRate(windowSeconds: number): number {
    const cutoff = Date.now() - windowSeconds * 1000;
    const recentCount = this.stats.recentErrors.filter(t => t > cutoff).length;

    // 清理过期记录
    this.stats.recentErrors = this.stats.recentErrors.filter(t => t > cutoff);

    if (this.stats.attempts === 0) return 0;
    return recentCount / Math.min(this.stats.attempts, 20); // 最多看最近 20 次
  }
}

// 使用示例
const retry = new AdaptiveRetry({
  maxRetries: 5,
  baseDelayMs: 1000,
  maxDelayMs: 60000,
  errorWindowSeconds: 60,
  failureRateThreshold: 0.5,
});

const result = await retry.execute(() =>
  client.messages.create({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 1024,
    messages: [{ role: 'user', content: '分析这段代码' }],
  })
);
```

---

## 21.3 断路器模式：保护你的系统

重试是好的，但**无限重试一个挂掉的服务**只会让你的系统也挂掉。断路器就是那个「别再试了，等服务恢复」的守护者。

### 21.3.1 断路器原理

断路器有三个状态：

```
         成功率恢复
     ┌─────────────────┐
     │                 │
     ▼                 │
  ┌──────┐  失败达阈值  ┌──────┐  超时后  ┌──────┐
  │ 关闭 │ ──────────→ │ 打开 │ ──────→ │半开 │
  │(正常) │             │(熔断) │          │(试探) │
  └──────┘             └──────┘          └──────┘
     ▲                                     │
     │           试探成功                    │
     └─────────────────────────────────────┘
                                        试探失败
                                     ┌──────────┐
                                     │ 回到打开  │
                                     └──────────┘
```

- **关闭（Closed）**：正常工作，记录失败次数
- **打开（Open）**：熔断状态，直接拒绝请求，不调用 API
- **半开（Half-Open）**：试探状态，放少量请求通过，成功则关闭，失败则重新打开

### 21.3.2 完整的断路器实现

```typescript
// circuit-breaker.ts — 生产级断路器

export enum CircuitState {
  CLOSED = 'closed',
  OPEN = 'open',
  HALF_OPEN = 'half_open',
}

interface CircuitBreakerOptions {
  /** 失败次数阈值，达到后打开断路器 */
  failureThreshold: number;
  /** 统计窗口时间（毫秒） */
  windowMs: number;
  /** 断路器打开后，多久进入半开状态（毫秒） */
  recoveryTimeoutMs: number;
  /** 半开状态下允许通过的试探请求数 */
  halfOpenMaxAttempts: number;
  /** 半开状态需要的成功次数才能关闭断路器 */
  halfOpenSuccessThreshold: number;
  /** 监听状态变化 */
  onStateChange?: (from: CircuitState, to: CircuitState) => void;
  /** 降级函数，断路器打开时调用 */
  fallback?: (error: Error) => Promise<any>;
}

class CircuitBreaker {
  private state: CircuitState = CircuitState.CLOSED;
  private failures: number[] = []; // 失败时间戳
  private halfOpenAttempts = 0;
  private halfOpenSuccesses = 0;
  private openedAt: number = 0;

  constructor(private options: CircuitBreakerOptions) {}

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    // 检查是否应该从打开转为半开
    if (this.state === CircuitState.OPEN) {
      if (Date.now() - this.openedAt >= this.options.recoveryTimeoutMs) {
        this.transitionTo(CircuitState.HALF_OPEN);
      } else {
        // 断路器打开，直接走降级
        if (this.options.fallback) {
          return this.options.fallback(
            new Error('Circuit breaker is OPEN')
          ) as Promise<T>;
        }
        throw new Error(
          `Circuit breaker is OPEN. Retry after ${Math.round(
            (this.options.recoveryTimeoutMs - (Date.now() - this.openedAt)) / 1000
          )}s`
        );
      }
    }

    // 半开状态：限制试探请求数
    if (this.state === CircuitState.HALF_OPEN) {
      if (this.halfOpenAttempts >= this.options.halfOpenMaxAttempts) {
        // 试探名额用完，走降级
        if (this.options.fallback) {
          return this.options.fallback(
            new Error('Circuit breaker HALF_OPEN: max attempts reached')
          ) as Promise<T>;
        }
        throw new Error('Circuit breaker HALF_OPEN: max probe attempts reached');
      }
      this.halfOpenAttempts++;
    }

    // 执行实际请求
    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  private onSuccess(): void {
    if (this.state === CircuitState.HALF_OPEN) {
      this.halfOpenSuccesses++;
      if (this.halfOpenSuccesses >= this.options.halfOpenSuccessThreshold) {
        this.transitionTo(CircuitState.CLOSED);
      }
    }
    // 关闭状态下的成功：清理失败计数
    this.failures = this.failures.filter(t => t > Date.now() - this.options.windowMs);
  }

  private onFailure(): void {
    this.failures.push(Date.now());

    // 清理窗口外的失败记录
    const cutoff = Date.now() - this.options.windowMs;
    this.failures = this.failures.filter(t => t > cutoff);

    if (this.state === CircuitState.HALF_OPEN) {
      // 半开状态下失败，重新打开
      this.transitionTo(CircuitState.OPEN);
    } else if (this.state === CircuitState.CLOSED) {
      // 关闭状态下失败，检查是否达到阈值
      if (this.failures.length >= this.options.failureThreshold) {
        this.transitionTo(CircuitState.OPEN);
      }
    }
  }

  private transitionTo(newState: CircuitState): void {
    const oldState = this.state;
    this.state = newState;

    if (newState === CircuitState.OPEN) {
      this.openedAt = Date.now();
      this.halfOpenAttempts = 0;
      this.halfOpenSuccesses = 0;
    }

    if (newState === CircuitState.CLOSED) {
      this.failures = [];
      this.halfOpenAttempts = 0;
      this.halfOpenSuccesses = 0;
    }

    if (newState === CircuitState.HALF_OPEN) {
      this.halfOpenAttempts = 0;
      this.halfOpenSuccesses = 0;
    }

    this.options.onStateChange?.(oldState, newState);
    console.log(`🔄 断路器状态: ${oldState} → ${newState}`);
  }

  /** 获取当前状态（用于监控） */
  getState(): CircuitState {
    return this.state;
  }

  /** 获取统计信息（用于监控） */
  getStats() {
    return {
      state: this.state,
      recentFailures: this.failures.length,
      halfOpenAttempts: this.halfOpenAttempts,
      halfOpenSuccesses: this.halfOpenSuccesses,
      openedAt: this.openedAt || null,
    };
  }
}

// 使用示例：为 API 调用加断路器
const circuitBreaker = new CircuitBreaker({
  failureThreshold: 5,       // 5 次失败后熔断
  windowMs: 60000,           // 统计 60 秒窗口
  recoveryTimeoutMs: 30000,  // 30 秒后进入半开
  halfOpenMaxAttempts: 3,    // 允许 3 次试探
  halfOpenSuccessThreshold: 2, // 2 次成功才关闭
  onStateChange: (from, to) => {
    console.log(`⚠️ 断路器: ${from} → ${to}`);
    // 这里可以接入告警系统
  },
  fallback: async (error) => {
    console.log('🪫 降级响应：使用缓存结果');
    return { content: '抱歉，服务暂时不可用，请稍后再试。' };
  },
});

// 用断路器包装 API 调用
try {
  const result = await circuitBreaker.execute(() =>
    client.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 1024,
      messages: [{ role: 'user', content: '你好' }],
    })
  );
} catch (error) {
  console.error('请求失败且断路器已降级:', error);
}
```

---

## 21.4 错误聚合与降级策略

当错误密集出现时，不能只靠重试。你需要**降级**——用次优方案保证系统可用。

### 21.4.1 多模型降级

```typescript
// fallback-chain.ts — 模型降级链

interface ModelConfig {
  model: string;
  maxTokens: number;
  priority: number; // 越小优先级越高
}

class ModelFallbackChain {
  private models: ModelConfig[];

  constructor(models: ModelConfig[]) {
    // 按 priority 排序
    this.models = [...models].sort((a, b) => a.priority - b.priority);
  }

  async chat(
    messages: Array<{ role: string; content: string }>,
    options?: {
      maxRetriesPerModel?: number;
      onFallback?: (from: string, to: string, error: Error) => void;
    }
  ): Promise<any> {
    const maxRetries = options?.maxRetriesPerModel ?? 1;
    const errors: Error[] = [];

    for (const modelConfig of this.models) {
      for (let attempt = 0; attempt < maxRetries; attempt++) {
        try {
          const result = await client.messages.create({
            model: modelConfig.model,
            max_tokens: modelConfig.maxTokens,
            messages: messages as any,
          });

          if (attempt > 0) {
            console.log(`✅ 模型 ${modelConfig.model} 第 ${attempt + 1} 次尝试成功`);
          }

          return result;
        } catch (error) {
          errors.push(error as Error);
          const classified = new ErrorClassifier().classify(error);

          // 不可恢复的错误，直接跳到下一个模型
          if (classified.recoverability === 'permanent') {
            break; // 跳出内层循环，换模型
          }

          // 可恢复但已用完重试
          if (attempt === maxRetries - 1) {
            const nextModel = this.models[this.models.indexOf(modelConfig) + 1];
            if (nextModel) {
              options?.onFallback?.(modelConfig.model, nextModel.model, error as Error);
              console.log(
                `🔄 模型 ${modelConfig.model} 失败，降级到 ${nextModel.model}`
              );
            }
            break;
          }

          // 可恢复，等一会重试
          await sleep(classified.suggestedRetryDelay > 0 ? classified.suggestedRetryDelay : 2000);
        }
      }
    }

    // 所有模型都失败了
    throw new AggregateError(
      errors,
      `所有 ${this.models.length} 个模型均调用失败`
    );
  }
}

// 使用示例
const fallbackChain = new ModelFallbackChain([
  { model: 'claude-sonnet-4-20250514', maxTokens: 4096, priority: 1 },
  { model: 'claude-3-5-haiku-20241022', maxTokens: 4096, priority: 2 },
]);

const result = await fallbackChain.chat(
  [{ role: 'user', content: '解释量子计算的基本原理' }],
  {
    maxRetriesPerModel: 2,
    onFallback: (from, to, error) => {
      console.log(`⚠️ ${from} → ${to}，原因: ${error.message}`);
    },
  }
);
```

### 21.4.2 缓存降级

```typescript
// cache-fallback.ts — 缓存降级

interface CacheEntry<T> {
  data: T;
  timestamp: number;
  query: string;
}

class CacheFallback {
  private cache = new Map<string, CacheEntry<any>>();
  private maxAgeMs: number;

  constructor(maxAgeMs: number = 3600000) { // 默认 1 小时
    this.maxAgeMs = maxAgeMs;
  }

  /**
   * 先调 API，失败则用缓存
   */
  async get<T>(
    key: string,
    fetcher: () => Promise<T>,
    options?: {
      /** 是否接受过期缓存 */
      acceptStale?: boolean;
      /** 过期缓存的最大年龄（毫秒） */
      maxStaleAgeMs?: number;
    }
  ): Promise<T> {
    try {
      const result = await fetcher();
      this.cache.set(key, {
        data: result,
        timestamp: Date.now(),
        query: key,
      });
      return result;
    } catch (error) {
      const entry = this.cache.get(key);
      if (!entry) throw error;

      const age = Date.now() - entry.timestamp;
      const isStale = age > this.maxAgeMs;

      if (!isStale) {
        console.log(`📦 API 失败，使用缓存（${Math.round(age / 1000)}s 前）`);
        return entry.data as T;
      }

      if (options?.acceptStale) {
        const maxStale = options.maxStaleAgeMs ?? this.maxAgeMs * 24; // 默认允许过期 24 倍时间
        if (age < maxStale) {
          console.log(
            `📦 API 失败，使用过期缓存（${Math.round(age / 60000)}min 前）`
          );
          return entry.data as T;
        }
      }

      throw error;
    }
  }
}

// 使用示例
const cacheFallback = new CacheFallback(1800000); // 30 分钟缓存

const answer = await cacheFallback.get(
  'python-basics',
  () => client.messages.create({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 1024,
    messages: [{ role: 'user', content: 'Python 基础入门' }],
  }),
  { acceptStale: true, maxStaleAgeMs: 86400000 } // 允许 24 小时内的过期缓存
);
```

---

## 21.5 ResilientClient：把一切组合起来

现在我们把错误分类、重试、断路器、降级全部整合成一个**韧性客户端**：

```typescript
// resilient-client.ts — 生产级韧性客户端

import Anthropic from '@anthropic-ai/sdk';
import { ErrorClassifier, ClassifiedError, ErrorCategory } from './error-classifier';
import { CircuitBreaker, CircuitState } from './circuit-breaker';

export interface ResilientClientOptions {
  /** Anthropic API Key */
  apiKey: string;
  /** 默认模型 */
  defaultModel?: string;
  /** 重试配置 */
  retry?: {
    maxRetries?: number;
    baseDelayMs?: number;
    maxDelayMs?: number;
  };
  /** 断路器配置 */
  circuitBreaker?: {
    failureThreshold?: number;
    windowMs?: number;
    recoveryTimeoutMs?: number;
  };
  /** 降级模型列表 */
  fallbackModels?: string[];
  /** 全局降级函数（断路器打开时调用） */
  fallback?: (error: Error) => Promise<any>;
  /** 错误事件回调 */
  onError?: (classified: ClassifiedError) => void;
}

export class ResilientClient {
  private client: Anthropic;
  private classifier: ErrorClassifier;
  private circuitBreaker: CircuitBreaker;
  private options: Required<ResilientClientOptions>;

  constructor(options: ResilientClientOptions) {
    this.client = new Anthropic({ apiKey: options.apiKey });
    this.classifier = new ErrorClassifier();

    // 填充默认值
    this.options = {
      apiKey: options.apiKey,
      defaultModel: options.defaultModel ?? 'claude-sonnet-4-20250514',
      retry: {
        maxRetries: options.retry?.maxRetries ?? 3,
        baseDelayMs: options.retry?.baseDelayMs ?? 1000,
        maxDelayMs: options.retry?.maxDelayMs ?? 60000,
      },
      circuitBreaker: {
        failureThreshold: options.circuitBreaker?.failureThreshold ?? 5,
        windowMs: options.circuitBreaker?.windowMs ?? 60000,
        recoveryTimeoutMs: options.circuitBreaker?.recoveryTimeoutMs ?? 30000,
      },
      fallbackModels: options.fallbackModels ?? [],
      fallback: options.fallback ?? (async () => {
        throw new Error('All fallbacks exhausted');
      }),
      onError: options.onError ?? (() => {}),
    };

    this.circuitBreaker = new CircuitBreaker({
      failureThreshold: this.options.circuitBreaker.failureThreshold,
      windowMs: this.options.circuitBreaker.windowMs,
      recoveryTimeoutMs: this.options.circuitBreaker.recoveryTimeoutMs,
      halfOpenMaxAttempts: 3,
      halfOpenSuccessThreshold: 2,
      onStateChange: (from, to) => {
        console.log(`🔄 断路器: ${from} → ${to}`);
      },
      fallback: this.options.fallback,
    });
  }

  /**
   * 带完整韧性策略的 API 调用
   */
  async chat(
    messages: Array<{ role: 'user' | 'assistant'; content: string }>,
    options?: {
      model?: string;
      maxTokens?: number;
      system?: string;
      tools?: any[];
    }
  ): Promise<Anthropic.Message> {
    const model = options?.model ?? this.options.defaultModel;
    const allModels = [model, ...this.options.fallbackModels];

    let lastError: Error | null = null;

    for (const currentModel of allModels) {
      try {
        return await this.circuitBreaker.execute(() =>
          this.callWithRetry({
            model: currentModel,
            messages,
            max_tokens: options?.maxTokens ?? 4096,
            system: options?.system,
            tools: options?.tools,
          })
        );
      } catch (error) {
        lastError = error as Error;
        const classified = this.classifier.classify(error);
        this.options.onError(classified);

        // 不可恢复，换模型试试
        if (classified.recoverability === 'permanent') {
          const nextIdx = allModels.indexOf(currentModel) + 1;
          if (nextIdx < allModels.length) {
            console.log(
              `🔄 模型 ${currentModel} 不可恢复错误，尝试 ${allModels[nextIdx]}`
            );
          }
          continue;
        }

        // 断路器降级
        if (classified.category === ErrorCategory.RATE_LIMIT && 
            this.circuitBreaker.getState() === CircuitState.OPEN) {
          continue; // 尝试下一个模型
        }

        // 其他错误，重试已用完
        continue;
      }
    }

    throw lastError ?? new Error('All models failed');
  }

  /**
   * 带重试的单次 API 调用
   */
  private async callWithRetry(params: any): Promise<Anthropic.Message> {
    const { maxRetries, baseDelayMs, maxDelayMs } = this.options.retry;
    let lastError: unknown;

    for (let attempt = 0; attempt <= maxRetries; attempt++) {
      try {
        return await this.client.messages.create(params);
      } catch (error) {
        lastError = error;
        const classified = this.classifier.classify(error);

        // 不可恢复的错误，直接抛
        if (classified.recoverability === 'permanent') {
          throw error;
        }

        // 最后一次尝试，不等了
        if (attempt === maxRetries) {
          throw error;
        }

        // 使用分类器建议的延迟，或指数退避
        let delay = classified.suggestedRetryDelay > 0
          ? classified.suggestedRetryDelay
          : baseDelayMs * Math.pow(2, attempt);

        delay = Math.min(delay, maxDelayMs);
        delay *= 0.5 + Math.random() * 0.5; // 抖动

        console.log(
          `⏳ 重试 ${attempt + 1}/${maxRetries}，` +
          `${currentModel} → 等待 ${Math.round(delay)}ms ` +
          `(${classified.category})`
        );

        await sleep(delay);
      }
    }

    throw lastError;
  }

  /** 获取断路器状态（用于健康检查） */
  getCircuitBreakerState(): CircuitState {
    return this.circuitBreaker.getState();
  }

  /** 获取断路器统计（用于监控） */
  getCircuitBreakerStats() {
    return this.circuitBreaker.getStats();
  }
}

function sleep(ms: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, ms));
}
```

### 使用 ResilientClient

```typescript
import { ResilientClient } from './resilient-client';

// 1. 创建客户端
const client = new ResilientClient({
  apiKey: process.env.ANTHROPIC_API_KEY!,
  defaultModel: 'claude-sonnet-4-20250514',
  retry: {
    maxRetries: 3,
    baseDelayMs: 1000,
    maxDelayMs: 30000,
  },
  circuitBreaker: {
    failureThreshold: 5,
    windowMs: 60000,
    recoveryTimeoutMs: 30000,
  },
  fallbackModels: ['claude-3-5-haiku-20241022'],
  fallback: async () => ({
    content: [{ type: 'text', text: '服务暂时不可用，请稍后再试。' }],
  }),
  onError: (classified) => {
    if (classified.needsAlert) {
      console.error(`🚨 需要告警: ${classified.description}`);
    }
  },
});

// 2. 使用
try {
  const response = await client.chat(
    [{ role: 'user', content: '帮我写一个快排算法' }],
    { maxTokens: 2048 }
  );
  console.log(response.content[0].text);
} catch (error) {
  console.error('所有策略均失败:', error);
}

// 3. 健康检查（可接入 /health 端点）
app.get('/health', (req, res) => {
  const state = client.getCircuitBreakerState();
  const stats = client.getCircuitBreakerStats();
  res.json({
    status: state === CircuitState.CLOSED ? 'healthy' : 'degraded',
    circuitBreaker: stats,
  });
});
```

---

## 21.6 实战案例：智能重试管理器

把所有内容整合成一个可以**直接用在生产环境**的智能重试管理器：

```typescript
// smart-retry-manager.ts — 一站式错误处理方案

import { ErrorClassifier, ClassifiedError, ErrorCategory, ErrorRecoverability } from './error-classifier';
import { CircuitBreaker, CircuitState } from './circuit-breaker';

interface SmartRetryManagerOptions {
  /** 默认最大重试次数 */
  maxRetries?: number;
  /** 默认基础延迟 */
  baseDelayMs?: number;
  /** 默认最大延迟 */
  maxDelayMs?: number;
  /** 断路器配置 */
  circuitBreaker?: {
    failureThreshold?: number;
    windowMs?: number;
    recoveryTimeoutMs?: number;
  };
  /** 模型降级链 */
  modelChain?: Array<{ model: string; maxTokens?: number }>;
  /** 全局降级函数 */
  fallback?: (error: ClassifiedError) => Promise<any>;
  /** 错误回调 */
  onError?: (classified: ClassifiedError, context: string) => void;
  /** 重试前回调（可用于修改请求） */
  beforeRetry?: (classified: ClassifiedError, attempt: number) => Promise<void>;
}

export class SmartRetryManager {
  private classifier: ErrorClassifier;
  private circuitBreakers: Map<string, CircuitBreaker> = new Map();
  private options: Required<SmartRetryManagerOptions>;

  // 统计数据
  private stats = {
    totalCalls: 0,
    successCalls: 0,
    failedCalls: 0,
    retriedCalls: 0,
    fallbackCalls: 0,
    circuitBreakerTrips: 0,
  };

  constructor(options: SmartRetryManagerOptions = {}) {
    this.classifier = new ErrorClassifier();
    this.options = {
      maxRetries: options.maxRetries ?? 3,
      baseDelayMs: options.baseDelayMs ?? 1000,
      maxDelayMs: options.maxDelayMs ?? 60000,
      circuitBreaker: {
        failureThreshold: options.circuitBreaker?.failureThreshold ?? 5,
        windowMs: options.circuitBreaker?.windowMs ?? 60000,
        recoveryTimeoutMs: options.circuitBreaker?.recoveryTimeoutMs ?? 30000,
      },
      modelChain: options.modelChain ?? [],
      fallback: options.fallback ?? (async () => {
        throw new Error('No fallback configured');
      }),
      onError: options.onError ?? (() => {}),
      beforeRetry: options.beforeRetry ?? (async () => {}),
    };
  }

  /**
   * 执行一个带完整韧性策略的异步操作
   */
  async execute<T>(
    fn: () => Promise<T>,
    context?: string
  ): Promise<T> {
    this.stats.totalCalls++;
    const ctx = context ?? 'default';
    const breaker = this.getOrCreateCircuitBreaker(ctx);

    // 检查断路器
    if (breaker.getState() === CircuitState.OPEN) {
      this.stats.fallbackCalls++;
      const classified: ClassifiedError = {
        original: new Error('Circuit breaker open'),
        category: ErrorCategory.OVERLOAD,
        severity: { LOW: 0, MEDIUM: 1, HIGH: 2, CRITICAL: 3 } as any,
        recoverability: ErrorRecoverability.CONDITIONAL,
        suggestedRetryDelay: -1,
        description: `断路器 [${ctx}] 已打开，请求被熔断`,
        needsAlert: false,
      };
      return this.options.fallback(classified) as Promise<T>;
    }

    let lastClassified: ClassifiedError | null = null;

    for (let attempt = 0; attempt <= this.options.maxRetries; attempt++) {
      try {
        const result = await breaker.execute(fn);
        this.stats.successCalls++;
        return result;
      } catch (error) {
        const classified = this.classifier.classify(error);
        lastClassified = classified;
        this.options.onError(classified, ctx);

        // 不可恢复，直接失败
        if (classified.recoverability === 'permanent') {
          this.stats.failedCalls++;
          throw error;
        }

        // 最后一次尝试
        if (attempt === this.options.maxRetries) {
          break;
        }

        this.stats.retriedCalls++;

        // 重试前回调（如：截断上下文）
        await this.options.beforeRetry(classified, attempt);

        // 计算延迟
        let delay = classified.suggestedRetryDelay > 0
          ? classified.suggestedRetryDelay
          : this.options.baseDelayMs * Math.pow(2, attempt);
        delay = Math.min(delay, this.options.maxDelayMs);
        delay *= 0.5 + Math.random() * 0.5;

        console.log(
          `⏳ [${ctx}] 重试 ${attempt + 1}/${this.options.maxRetries}，` +
          `等待 ${Math.round(delay)}ms (${classified.category})`
        );

        await new Promise(r => setTimeout(r, delay));
      }
    }

    // 所有重试失败，走降级
    this.stats.failedCalls++;
    if (lastClassified) {
      return this.options.fallback(lastClassified) as Promise<T>;
    }
    throw new Error('All retries exhausted');
  }

  private getOrCreateCircuitBreaker(context: string): CircuitBreaker {
    if (!this.circuitBreakers.has(context)) {
      const cb = new CircuitBreaker({
        failureThreshold: this.options.circuitBreaker.failureThreshold,
        windowMs: this.options.circuitBreaker.windowMs,
        recoveryTimeoutMs: this.options.circuitBreaker.recoveryTimeoutMs,
        halfOpenMaxAttempts: 3,
        halfOpenSuccessThreshold: 2,
        onStateChange: (from, to) => {
          if (to === CircuitState.OPEN) {
            this.stats.circuitBreakerTrips++;
          }
          console.log(`🔄 断路器 [${context}]: ${from} → ${to}`);
        },
      });
      this.circuitBreakers.set(context, cb);
    }
    return this.circuitBreakers.get(context)!;
  }

  /** 获取统计信息 */
  getStats() {
    return { ...this.stats };
  }
}

// ==================== 完整使用示例 ====================

async function demo() {
  const manager = new SmartRetryManager({
    maxRetries: 3,
    baseDelayMs: 1000,
    maxDelayMs: 30000,
    circuitBreaker: {
      failureThreshold: 5,
      windowMs: 60000,
      recoveryTimeoutMs: 30000,
    },
    modelChain: [
      { model: 'claude-sonnet-4-20250514', maxTokens: 4096 },
      { model: 'claude-3-5-haiku-20241022', maxTokens: 4096 },
    ],
    fallback: async (classified) => {
      console.log(`🪫 降级: ${classified.description}`);
      return { content: '服务暂时不可用，请稍后再试。' };
    },
    onError: (classified, context) => {
      if (classified.needsAlert) {
        console.error(`🚨 [${context}] ${classified.description}`);
      }
    },
    beforeRetry: async (classified, attempt) => {
      // 上下文过长时，自动截断
      if (classified.category === ErrorCategory.CONTEXT_LENGTH) {
        console.log('✂️ 上下文过长，尝试截断...');
        // 这里可以修改请求参数
      }
    },
  });

  // 执行 API 调用
  const result = await manager.execute(
    () => client.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 1024,
      messages: [{ role: 'user', content: 'Hello!' }],
    }),
    'chat-api' // 上下文标识
  );

  console.log('结果:', result);
  console.log('统计:', manager.getStats());
}
```

---

## 21.7 错误处理最佳实践清单

| # | 实践 | 说明 |
|---|------|------|
| 1 | **分类优先** | 先分类再处理，不要对所有错误一刀切 |
| 2 | **只重试暂时性错误** | 429、5xx、网络错误可重试；4xx（非 429）不可重试 |
| 3 | **用指数退避 + 抖动** | 固定间隔会造成惊群，纯指数退避会同步 |
| 4 | **尊重 Retry-After** | 429 的 Retry-After 头是最准确的等待时间 |
| 5 | **设重试上限** | 永远不要无限重试，设 maxRetries |
| 6 | **加断路器** | 超过阈值就熔断，别把下游打挂 |
| 7 | **准备降级** | 断路器打开时要有 Plan B（缓存/轻量模型） |
| 8 | **记录错误分类** | 哪类错误最多？指导优化方向 |
| 9 | **区分上下文** | 不同 API/模型用不同断路器，别一锅端 |
| 10 | **可观测** | 断路器状态、错误率、重试次数都要能看到 |

---

## 小结

错误处理是生产级 SDK 应用的**生命线**。记住这个递进关系：

```
错误分类 → 合理重试 → 断路器保护 → 降级兜底
```

1. **分类**：不是所有错误都值得重试，先搞清楚错在哪
2. **重试**：固定间隔（限流）、指数退避（服务端错误）、自适应（长期运行）
3. **断路器**：保护自己和下游，超过阈值就熔断
4. **降级**：缓存、轻量模型、礼貌提示——总比 500 好

下一章，我们聊**缓存策略与性能调优**——如何让重复查询几乎零成本。

---

> 💡 **老三提醒**：错误处理代码看似枯燥，但它决定了你凌晨 3 点能不能睡个好觉。写得越扎实，觉睡得越香。
