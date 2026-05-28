# 第25章：日志与监控系统集成

> 学会构建企业级可观测性体系，让你的 Claude Code SDK 应用"看得见、摸得着"。

## 25.1 为什么需要可观测性？

在生产环境中运行 Claude Code SDK 应用时，"出问题了我怎么知道？"是每个团队必须回答的问题。可观测性（Observability）让你能够：

1. **快速定位问题** - 当 API 响应变慢或失败时，能迅速找到根因
2. **性能持续优化** - 了解 Token 消耗、延迟分布、成本瓶颈
3. **容量规划** - 预测何时需要扩展、何时需要优化
4. **合规审计** - 满足企业的审计和安全要求

### 可观测性三支柱

现代可观测性由三大信号组成：

```
┌─────────────────────────────────────────────────────────────┐
│                    Observability 三支柱                      │
├─────────────┬─────────────────┬─────────────────────────────┤
│   Metrics   │     Logs        │         Traces              │
│   指标      │     日志         │         追踪                │
├─────────────┼─────────────────┼─────────────────────────────┤
│ 聚合数据    │ 离散事件记录     │ 请求链路追踪                │
│ "多少"      │ "发生了什么"     │ "经过哪里"                  │
│             │                 │                             │
│ 示例：      │ 示例：           │ 示例：                      │
│ - 请求速率  │ - 错误日志       │ - API调用链                 │
│ - 延迟分布  │ - 调试信息       │ - 跨服务依赖                │
│ - Token消耗 │ - 审计记录       │ - 瓶颈定位                  │
└─────────────┴─────────────────┴─────────────────────────────┘
```

## 25.2 结构化日志：让日志可查询

### 25.2.1 传统日志的问题

传统的 `console.log` 日志存在以下问题：

```javascript
// ❌ 传统日志 - 难以查询和分析
console.log('User requested translation');
console.log('API call took 2.5 seconds');
console.log('Error: rate limit exceeded');

// 问题：
// 1. 无法按字段过滤（如按用户ID、按错误类型）
// 2. 时间格式不统一
// 3. 缺少上下文信息
// 4. 无法聚合统计
```

### 25.2.2 结构化日志最佳实践

使用 **Pino** 或 **Winston** 实现结构化日志：

```javascript
// 安装 Pino
// npm install pino pino-pretty

import pino from 'pino';

// 创建日志实例
const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  transport: process.env.NODE_ENV !== 'production' 
    ? { target: 'pino-pretty', options: { colorize: true } }
    : undefined,
  // 添加默认字段
  base: {
    service: 'claude-sdk-app',
    version: process.env.npm_package_version,
    env: process.env.NODE_ENV
  },
  // 时间戳格式
  timestamp: pino.stdTimeFunctions.isoTime,
  // 格式化错误对象
  formatters: {
    bindings: (bindings) => ({ pid: bindings.pid, host: bindings.hostname })
  }
});

export default logger;
```

### 25.2.3 Claude SDK 专用日志封装

```javascript
// claude-logger.js - Claude SDK 专用日志封装
import logger from './logger.js';
import { nanoid } from 'nanoid';

/**
 * Claude SDK 专用日志记录器
 * 自动记录请求耗时、Token 消耗、成本等关键指标
 */
export class ClaudeLogger {
  constructor(options = {}) {
    this.logger = options.logger || logger;
    this.userId = options.userId;
    this.sessionId = options.sessionId;
  }

  /**
   * 生成追踪ID
   */
  generateTraceId() {
    return `trace_${nanoid(16)}`;
  }

  /**
   * 记录 API 请求开始
   */
  logRequest(params) {
    const traceId = this.generateTraceId();
    const startTime = Date.now();
    
    this.logger.info({
      event: 'claude_request_start',
      traceId,
      model: params.model,
      messageCount: params.messages?.length || 0,
      hasTools: !!params.tools,
      maxTokens: params.max_tokens,
      userId: this.userId,
      sessionId: this.sessionId
    }, `开始 Claude API 调用: ${params.model}`);
    
    return { traceId, startTime };
  }

  /**
   * 记录 API 请求成功
   */
  logSuccess(traceId, startTime, response) {
    const duration = Date.now() - startTime;
    const usage = response.usage || {};
    
    this.logger.info({
      event: 'claude_request_success',
      traceId,
      duration,
      model: response.model,
      // Token 使用
      inputTokens: usage.input_tokens,
      outputTokens: usage.output_tokens,
      totalTokens: (usage.input_tokens || 0) + (usage.output_tokens || 0),
      // 成本估算（按 Claude 3.5 Sonnet 定价）
      estimatedCost: this._estimateCost(usage, response.model),
      // 响应信息
      stopReason: response.stop_reason,
      hasToolUse: response.content?.some(c => c.type === 'tool_use'),
      userId: this.userId,
      sessionId: this.sessionId
    }, `Claude API 调用成功: ${duration}ms, ${usage.input_tokens || 0}+${usage.output_tokens || 0} tokens`);
  }

  /**
   * 记录 API 请求失败
   */
  logError(traceId, startTime, error) {
    const duration = Date.now() - startTime;
    
    this.logger.error({
      event: 'claude_request_error',
      traceId,
      duration,
      errorCode: error.status || error.code,
      errorMessage: error.message,
      errorType: error.constructor.name,
      // 错误详情
      isRetryable: this._isRetryableError(error),
      isRateLimit: error.status === 429,
      userId: this.userId,
      sessionId: this.sessionId,
      // 错误堆栈（仅调试模式）
      stack: process.env.LOG_LEVEL === 'debug' ? error.stack : undefined
    }, `Claude API 调用失败: ${error.message}`);
  }

  /**
   * 记录流式响应事件
   */
  logStreamEvent(traceId, eventType, data = {}) {
    this.logger.debug({
      event: `claude_stream_${eventType}`,
      traceId,
      ...data,
      userId: this.userId,
      sessionId: this.sessionId
    }, `流式事件: ${eventType}`);
  }

  /**
   * 估算成本（基于 Claude 3.5 Sonnet 定价）
   */
  _estimateCost(usage, model) {
    const pricing = {
      'claude-3-5-sonnet-20241022': { input: 3, output: 15 }, // 每百万token
      'claude-3-opus-20240229': { input: 15, output: 75 },
      'claude-3-haiku-20240307': { input: 0.25, output: 1.25 }
    };
    
    const rate = pricing[model] || pricing['claude-3-5-sonnet-20241022'];
    const inputCost = (usage.input_tokens / 1_000_000) * rate.input;
    const outputCost = (usage.output_tokens / 1_000_000) * rate.output;
    
    return Number((inputCost + outputCost).toFixed(6));
  }

  /**
   * 判断是否可重试错误
   */
  _isRetryableError(error) {
    const retryableStatuses = [429, 500, 502, 503, 504];
    return retryableStatuses.includes(error.status);
  }
}

// 使用示例
const claudeLogger = new ClaudeLogger({ 
  userId: 'user_123', 
  sessionId: 'session_abc' 
);

// 在 API 调用中使用
const { traceId, startTime } = claudeLogger.logRequest({
  model: 'claude-3-5-sonnet-20241022',
  messages: [{ role: 'user', content: 'Hello' }],
  tools: [{ name: 'search' }]
});

try {
  const response = await client.messages.create(params);
  claudeLogger.logSuccess(traceId, startTime, response);
} catch (error) {
  claudeLogger.logError(traceId, startTime, error);
  throw error;
}
```

### 25.2.4 日志输出示例

```json
// 标准日志输出格式
{
  "level": 30,
  "time": "2026-05-27T15:30:45.123Z",
  "pid": 12345,
  "host": "app-server-01",
  "service": "claude-sdk-app",
  "version": "1.2.3",
  "env": "production",
  "event": "claude_request_success",
  "traceId": "trace_a1b2c3d4e5f6g7h8",
  "duration": 2345,
  "model": "claude-3-5-sonnet-20241022",
  "inputTokens": 150,
  "outputTokens": 320,
  "totalTokens": 470,
  "estimatedCost": 0.00525,
  "stopReason": "end_turn",
  "hasToolUse": false,
  "userId": "user_123",
  "sessionId": "session_abc"
}
```

## 25.3 OpenTelemetry：统一可观测性标准

### 25.3.1 OpenTelemetry 简介

OpenTelemetry（OTel）是 CNCF 的毕业项目，提供了厂商中立的可观测性标准：

```
┌─────────────────────────────────────────────────────────────────┐
│                    OpenTelemetry 架构                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐         │
│  │    API      │    │    SDK      │    │ Collector   │         │
│  │  (接口层)    │───▶│  (实现层)   │───▶│  (收集层)   │         │
│  └─────────────┘    └─────────────┘    └──────┬──────┘         │
│                                                │                │
│              ┌─────────────────────────────────┼───────────┐   │
│              │                                 │           │   │
│              ▼                                 ▼           ▼   │
│     ┌─────────────┐                   ┌─────────────┐ ┌─────┐ │
│     │  Jaeger     │                   │ Prometheus  │ │ELK  │ │
│     │  (Traces)   │                   │ (Metrics)   │ │(Log)│ │
│     └─────────────┘                   └─────────────┘ └─────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### 25.3.2 快速集成 OpenTelemetry

```bash
# 安装 OpenTelemetry 依赖
npm install @opentelemetry/api \
            @opentelemetry/sdk-node \
            @opentelemetry/auto-instrumentations-node \
            @opentelemetry/exporter-trace-otlp-http \
            @opentelemetry/exporter-metrics-otlp-http \
            @opentelemetry/instrumentation-http \
            @opentelemetry/instrumentation-fetch
```

### 25.3.3 初始化 OpenTelemetry

```javascript
// telemetry.js - OpenTelemetry 初始化配置
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { OTLPMetricExporter } from '@opentelemetry/exporter-metrics-otlp-http';
import { PeriodicExportingMetricReader } from '@opentelemetry/sdk-metrics';
import { Resource } from '@opentelemetry/resources';
import { SemanticResourceAttributes } from '@opentelemetry/semantic-conventions';
import { trace, metrics } from '@opentelemetry/api';

// 创建资源标识
const resource = new Resource({
  [SemanticResourceAttributes.SERVICE_NAME]: process.env.OTEL_SERVICE_NAME || 'claude-sdk-app',
  [SemanticResourceAttributes.SERVICE_VERSION]: process.env.npm_package_version || '1.0.0',
  [SemanticResourceAttributes.DEPLOYMENT_ENVIRONMENT]: process.env.NODE_ENV || 'development',
});

// 创建 Trace 导出器
const traceExporter = new OTLPTraceExporter({
  url: process.env.OTEL_EXPORTER_OTLP_TRACES_ENDPOINT || 'http://localhost:4318/v1/traces',
  headers: {
    'api-key': process.env.OTEL_API_KEY || '',
  },
});

// 创建 Metrics 导出器
const metricReader = new PeriodicExportingMetricReader({
  exporter: new OTLPMetricExporter({
    url: process.env.OTEL_EXPORTER_OTLP_METRICS_ENDPOINT || 'http://localhost:4318/v1/metrics',
  }),
  exportIntervalMillis: 60000, // 每分钟导出一次
});

// 初始化 SDK
const sdk = new NodeSDK({
  resource,
  traceExporter,
  metricReader,
  instrumentations: [
    getNodeAutoInstrumentations({
      // 启用 HTTP 自动追踪
      '@opentelemetry/instrumentation-http': {
        enabled: true,
      },
      // 启用 Fetch 自动追踪
      '@opentelemetry/instrumentation-fetch': {
        enabled: true,
      },
    }),
  ],
});

// 启动 SDK
export async function initTelemetry() {
  try {
    await sdk.start();
    console.log('✅ OpenTelemetry initialized successfully');
  } catch (error) {
    console.error('❌ Failed to initialize OpenTelemetry:', error);
  }
}

// 关闭 SDK（优雅退出）
export async function shutdownTelemetry() {
  try {
    await sdk.shutdown();
    console.log('✅ OpenTelemetry shutdown successfully');
  } catch (error) {
    console.error('❌ Failed to shutdown OpenTelemetry:', error);
  }
}

// 导出追踪器和计量器
export const tracer = trace.getTracer('claude-sdk-app');
export const meter = metrics.getMeter('claude-sdk-app');

// 在应用启动时调用
// initTelemetry();
// 在应用关闭时调用
// process.on('SIGTERM', shutdownTelemetry);
```

### 25.3.4 Claude SDK 自定义 Span

```javascript
// claude-telemetry.js - Claude SDK 遥测封装
import { tracer, meter } from './telemetry.js';
import { SpanStatusCode } from '@opentelemetry/api';

/**
 * Claude SDK 遥测追踪器
 * 自动创建 Span 和 Metrics
 */
export class ClaudeTelemetry {
  constructor() {
    // 创建指标
    this.requestCounter = meter.createCounter('claude_requests_total', {
      description: 'Total number of Claude API requests',
      unit: '1',
    });

    this.tokenCounter = meter.createCounter('claude_tokens_total', {
      description: 'Total tokens consumed',
      unit: '1',
    });

    this.latencyHistogram = meter.createHistogram('claude_request_duration_ms', {
      description: 'Request latency in milliseconds',
      unit: 'ms',
    });

    this.costCounter = meter.createCounter('claude_cost_total', {
      description: 'Total estimated cost in USD',
      unit: 'USD',
    });

    this.activeRequests = meter.createUpDownCounter('claude_active_requests', {
      description: 'Number of active requests',
      unit: '1',
    });
  }

  /**
   * 追踪 Claude API 调用
   */
  async traceRequest(params, fn) {
    const startTime = Date.now();
    
    // 创建 Span
    return tracer.startActiveSpan('claude.api.request', {
      attributes: {
        'claude.model': params.model,
        'claude.max_tokens': params.max_tokens,
        'claude.has_tools': !!params.tools,
        'claude.message_count': params.messages?.length || 0,
      },
    }, async (span) => {
      this.activeRequests.add(1);
      
      try {
        // 执行实际请求
        const response = await fn();
        
        // 记录成功指标
        const duration = Date.now() - startTime;
        const usage = response.usage || {};
        
        span.setAttributes({
          'claude.input_tokens': usage.input_tokens || 0,
          'claude.output_tokens': usage.output_tokens || 0,
          'claude.total_tokens': (usage.input_tokens || 0) + (usage.output_tokens || 0),
          'claude.stop_reason': response.stop_reason,
          'claude.duration_ms': duration,
        });
        
        span.setStatus({ code: SpanStatusCode.OK });
        span.end();
        
        // 更新指标
        this.requestCounter.add(1, { model: params.model, status: 'success' });
        this.tokenCounter.add(usage.input_tokens || 0, { model: params.model, type: 'input' });
        this.tokenCounter.add(usage.output_tokens || 0, { model: params.model, type: 'output' });
        this.latencyHistogram.record(duration, { model: params.model });
        
        const cost = this._estimateCost(usage, params.model);
        this.costCounter.add(cost, { model: params.model });
        
        this.activeRequests.add(-1);
        
        return response;
      } catch (error) {
        // 记录失败
        span.recordException(error);
        span.setAttributes({
          'claude.error.type': error.constructor.name,
          'claude.error.code': error.status || error.code,
          'claude.error.message': error.message,
        });
        span.setStatus({ 
          code: SpanStatusCode.ERROR, 
          message: error.message 
        });
        span.end();
        
        this.requestCounter.add(1, { 
          model: params.model, 
          status: 'error',
          error_type: error.constructor.name 
        });
        this.activeRequests.add(-1);
        
        throw error;
      }
    });
  }

  /**
   * 追踪流式响应
   */
  async *traceStream(params, asyncIterator) {
    const startTime = Date.now();
    let inputTokens = 0;
    let outputTokens = 0;
    
    await tracer.startActiveSpan('claude.api.stream', {
      attributes: {
        'claude.model': params.model,
        'claude.streaming': true,
      },
    }, async (span) => {
      this.activeRequests.add(1);
      
      try {
        for await (const event of asyncIterator) {
          // 记录流式事件
          span.addEvent('claude.stream.event', {
            'claude.event.type': event.type,
          });
          
          // 累计 Token
          if (event.type === 'message_delta' && event.usage) {
            outputTokens = event.usage.output_tokens || 0;
          }
          if (event.type === 'message_start' && event.message?.usage) {
            inputTokens = event.message.usage.input_tokens || 0;
          }
          
          yield event;
        }
        
        const duration = Date.now() - startTime;
        
        span.setAttributes({
          'claude.input_tokens': inputTokens,
          'claude.output_tokens': outputTokens,
          'claude.duration_ms': duration,
        });
        span.setStatus({ code: SpanStatusCode.OK });
        span.end();
        
        // 更新指标
        this.requestCounter.add(1, { model: params.model, status: 'success', streaming: 'true' });
        this.tokenCounter.add(inputTokens, { model: params.model, type: 'input' });
        this.tokenCounter.add(outputTokens, { model: params.model, type: 'output' });
        this.latencyHistogram.record(duration, { model: params.model, streaming: 'true' });
        this.activeRequests.add(-1);
        
      } catch (error) {
        span.recordException(error);
        span.setStatus({ code: SpanStatusCode.ERROR, message: error.message });
        span.end();
        this.activeRequests.add(-1);
        throw error;
      }
    });
  }

  _estimateCost(usage, model) {
    const pricing = {
      'claude-3-5-sonnet-20241022': { input: 3, output: 15 },
      'claude-3-opus-20240229': { input: 15, output: 75 },
      'claude-3-haiku-20240307': { input: 0.25, output: 1.25 },
    };
    const rate = pricing[model] || pricing['claude-3-5-sonnet-20241022'];
    return ((usage.input_tokens / 1_000_000) * rate.input) + 
           ((usage.output_tokens / 1_000_000) * rate.output);
  }
}

// 使用示例
const telemetry = new ClaudeTelemetry();

// 追踪普通请求
const response = await telemetry.traceRequest({
  model: 'claude-3-5-sonnet-20241022',
  messages: [{ role: 'user', content: 'Hello' }],
}, async () => {
  return await client.messages.create(params);
});

// 追踪流式请求
const stream = await client.messages.stream(params);
for await (const event of telemetry.traceStream(params, stream)) {
  // 处理流式事件
}
```

## 25.4 Prometheus + Grafana：监控仪表盘

### 25.4.1 Prometheus 指标暴露

```javascript
// prometheus-metrics.js - Prometheus 指标暴露
import client from 'prom-client';

// 创建 Registry
const register = new client.Registry();

// 添加默认标签
register.setDefaultLabels({
  app: 'claude-sdk-app',
  env: process.env.NODE_ENV || 'development',
});

// Claude SDK 专用指标
const claudeMetrics = {
  // 请求计数器
  requestsTotal: new client.Counter({
    name: 'claude_requests_total',
    help: 'Total Claude API requests',
    labelNames: ['model', 'status', 'error_type'],
    registers: [register],
  }),

  // Token 使用计数器
  tokensTotal: new client.Counter({
    name: 'claude_tokens_total',
    help: 'Total tokens consumed',
    labelNames: ['model', 'type'], // type: input/output
    registers: [register],
  }),

  // 请求延迟直方图
  requestDuration: new client.Histogram({
    name: 'claude_request_duration_seconds',
    help: 'Request duration in seconds',
    labelNames: ['model', 'streaming'],
    buckets: [0.5, 1, 2.5, 5, 10, 30, 60, 120], // 秒
    registers: [register],
  }),

  // 成本累计
  costTotal: new client.Counter({
    name: 'claude_cost_total_usd',
    help: 'Total estimated cost in USD',
    labelNames: ['model'],
    registers: [register],
  }),

  // 活跃请求数
  activeRequests: new client.Gauge({
    name: 'claude_active_requests',
    help: 'Number of active requests',
    registers: [register],
  }),

  // 错误率
  errorRate: new client.Gauge({
    name: 'claude_error_rate',
    help: 'Current error rate (errors/minute)',
    labelNames: ['model'],
    registers: [register],
  }),

  // Token 消耗速率
  tokenRate: new client.Gauge({
    name: 'claude_token_rate_per_minute',
    help: 'Token consumption rate per minute',
    labelNames: ['model', 'type'],
    registers: [register],
  }),
};

// 启动默认指标收集（CPU、内存等）
client.collectDefaultMetrics({ register });

// Express 暴露指标端点
export function setupMetricsEndpoint(app) {
  app.get('/metrics', async (req, res) => {
    try {
      res.set('Content-Type', register.contentType);
      res.end(await register.metrics());
    } catch (err) {
      res.status(500).end(err.message);
    }
  });
}

export { claudeMetrics, register };
```

### 25.4.2 集成到 Claude SDK 客户端

```javascript
// monitored-client.js - 带监控的 Claude 客户端
import Anthropic from '@anthropic-ai/sdk';
import { claudeMetrics } from './prometheus-metrics.js';

export class MonitoredClaudeClient {
  constructor(options = {}) {
    this.client = new Anthropic(options);
    this.pricing = {
      'claude-3-5-sonnet-20241022': { input: 3, output: 15 },
      'claude-3-opus-20240229': { input: 15, output: 75 },
      'claude-3-haiku-20240307': { input: 0.25, output: 1.25 },
    };
  }

  /**
   * 监控普通请求
   */
  async createMessage(params) {
    const model = params.model || 'claude-3-5-sonnet-20241022';
    const timer = claudeMetrics.requestDuration.startTimer({ model, streaming: 'false' });
    
    claudeMetrics.activeRequests.inc();
    
    try {
      const response = await this.client.messages.create(params);
      
      // 记录成功指标
      claudeMetrics.requestsTotal.inc({ model, status: 'success' });
      
      const usage = response.usage || {};
      claudeMetrics.tokensTotal.inc({ model, type: 'input' }, usage.input_tokens || 0);
      claudeMetrics.tokensTotal.inc({ model, type: 'output' }, usage.output_tokens || 0);
      
      const cost = this._estimateCost(usage, model);
      claudeMetrics.costTotal.inc({ model }, cost);
      
      timer();
      claudeMetrics.activeRequests.dec();
      
      return response;
    } catch (error) {
      claudeMetrics.requestsTotal.inc({ 
        model, 
        status: 'error',
        error_type: error.constructor.name 
      });
      timer();
      claudeMetrics.activeRequests.dec();
      throw error;
    }
  }

  /**
   * 监控流式请求
   */
  async *streamMessage(params) {
    const model = params.model || 'claude-3-5-sonnet-20241022';
    const timer = claudeMetrics.requestDuration.startTimer({ model, streaming: 'true' });
    
    claudeMetrics.activeRequests.inc();
    
    let inputTokens = 0;
    let outputTokens = 0;
    let hasError = false;
    
    try {
      const stream = this.client.messages.stream(params);
      
      for await (const event of stream) {
        if (event.type === 'message_start' && event.message?.usage) {
          inputTokens = event.message.usage.input_tokens || 0;
        }
        if (event.type === 'message_delta' && event.usage) {
          outputTokens = event.usage.output_tokens || 0;
        }
        
        yield event;
      }
      
      // 记录成功
      claudeMetrics.requestsTotal.inc({ model, status: 'success' });
      claudeMetrics.tokensTotal.inc({ model, type: 'input' }, inputTokens);
      claudeMetrics.tokensTotal.inc({ model, type: 'output' }, outputTokens);
      
    } catch (error) {
      hasError = true;
      claudeMetrics.requestsTotal.inc({ 
        model, 
        status: 'error',
        error_type: error.constructor.name 
      });
      throw error;
    } finally {
      timer();
      claudeMetrics.activeRequests.dec();
      
      if (!hasError) {
        const cost = this._estimateCost(
          { input_tokens: inputTokens, output_tokens: outputTokens }, 
          model
        );
        claudeMetrics.costTotal.inc({ model }, cost);
      }
    }
  }

  _estimateCost(usage, model) {
    const rate = this.pricing[model] || this.pricing['claude-3-5-sonnet-20241022'];
    return ((usage.input_tokens / 1_000_000) * rate.input) + 
           ((usage.output_tokens / 1_000_000) * rate.output);
  }
}

// 使用示例
const client = new MonitoredClaudeClient({ apiKey: process.env.ANTHROPIC_API_KEY });

// 普通请求
const response = await client.createMessage({
  model: 'claude-3-5-sonnet-20241022',
  max_tokens: 1024,
  messages: [{ role: 'user', content: 'Hello!' }],
});

// 流式请求
for await (const event of client.streamMessage({
  model: 'claude-3-5-sonnet-20241022',
  max_tokens: 1024,
  messages: [{ role: 'user', content: 'Tell me a story' }],
})) {
  // 处理事件
}
```

### 25.4.3 Grafana 仪表盘配置

```yaml
# grafana-dashboard.yaml - Grafana 仪表盘配置
apiVersion: 1
providers:
  - name: 'Claude SDK Dashboard'
    folder: 'AI Services'
    type: file
    options:
      path: /var/lib/grafana/dashboards

# dashboard.json 内容概要：
# 1. 请求速率面板 (rate(claude_requests_total[5m]))
# 2. 延迟分布面板 (histogram_quantile)
# 3. Token 消耗面板 (rate(claude_tokens_total[1h]))
# 4. 成本累计面板 (claude_cost_total_usd)
# 5. 错误率面板 (rate(claude_requests_total{status="error"}[5m]))
# 6. 活跃请求面板 (claude_active_requests)
```

**Grafana 面板 PromQL 查询示例：**

```promql
# 1. 每分钟请求数
rate(claude_requests_total[1m]) * 60

# 2. P95 延迟
histogram_quantile(0.95, rate(claude_request_duration_seconds_bucket[5m]))

# 3. 每小时 Token 消耗
rate(claude_tokens_total[1h]) * 3600

# 4. 每小时成本
rate(claude_cost_total_usd[1h]) * 3600

# 5. 错误率
sum(rate(claude_requests_total{status="error"}[5m])) 
/ sum(rate(claude_requests_total[5m]))

# 6. 按模型分组的请求分布
sum by (model) (rate(claude_requests_total[5m]))
```

## 25.5 告警系统：主动发现问题

### 25.5.1 Prometheus 告警规则

```yaml
# prometheus-alerts.yaml - Claude SDK 告警规则
groups:
  - name: claude-sdk-alerts
    rules:
      # 高错误率告警
      - alert: ClaudeHighErrorRate
        expr: |
          sum(rate(claude_requests_total{status="error"}[5m])) 
          / sum(rate(claude_requests_total[5m])) > 0.05
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Claude API 错误率过高"
          description: "过去5分钟错误率超过5%，当前值: {{ $value | humanizePercentage }}"

      # 延迟过高告警
      - alert: ClaudeHighLatency
        expr: |
          histogram_quantile(0.95, rate(claude_request_duration_seconds_bucket[5m])) > 30
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Claude API 延迟过高"
          description: "P95延迟超过30秒，当前值: {{ $value | humanizeDuration }}"

      # Token 消耗异常
      - alert: ClaudeTokenSpike
        expr: |
          rate(claude_tokens_total[1h]) > 2 * rate(claude_tokens_total[24h])
        for: 30m
        labels:
          severity: info
        annotations:
          summary: "Token 消耗异常增长"
          description: "过去1小时Token消耗是24小时平均值的2倍以上"

      # 成本告警
      - alert: ClaudeCostWarning
        expr: rate(claude_cost_total_usd[1h]) * 24 * 30 > 1000
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "Claude API 月度成本预估过高"
          description: "按当前消耗速度，月度成本将超过$1000"

      # Rate Limit 频繁
      - alert: ClaudeRateLimitHit
        expr: |
          increase(claude_requests_total{error_type="RateLimitError"}[1h]) > 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "频繁触发 Rate Limit"
          description: "过去1小时触发Rate Limit超过10次"

      # 服务不可用
      - alert: ClaudeServiceDown
        expr: claude_active_requests == 0 and time() - claude_requests_total > 300
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Claude SDK 服务可能已停止"
          description: "过去5分钟没有任何API请求"
```

### 25.5.2 告警通知集成

```javascript
// alert-notifier.js - 告警通知处理器
import logger from './logger.js';

/**
 * 告警通知处理器
 * 支持多种通知渠道
 */
export class AlertNotifier {
  constructor(config = {}) {
    this.webhooks = config.webhooks || [];
    this.slackWebhook = config.slackWebhook;
    this.emailConfig = config.email;
    this.pagerDutyKey = config.pagerDutyKey;
  }

  /**
   * 处理告警
   */
  async handleAlert(alert) {
    logger.warn({ alert }, '收到告警');
    
    // 并行发送到所有配置的渠道
    const results = await Promise.allSettled([
      this.sendToSlack(alert),
      this.sendToWebhook(alert),
      this.sendEmail(alert),
      this.sendToPagerDuty(alert),
    ]);
    
    // 记录发送结果
    results.forEach((result, index) => {
      if (result.status === 'rejected') {
        logger.error({ error: result.reason, alert }, `告警发送失败: 渠道${index}`);
      }
    });
  }

  /**
   * 发送到 Slack
   */
  async sendToSlack(alert) {
    if (!this.slackWebhook) return;
    
    const color = this._getSlackColor(alert.labels.severity);
    
    const payload = {
      attachments: [{
        color,
        title: `🚨 ${alert.annotations.summary}`,
        text: alert.annotations.description,
        fields: [
          { title: 'Severity', value: alert.labels.severity, short: true },
          { title: 'Alert Name', value: alert.labels.alertname, short: true },
          { title: 'Time', value: new Date().toISOString(), short: true },
        ],
        footer: 'Claude SDK Monitor',
        ts: Math.floor(Date.now() / 1000),
      }],
    };
    
    const response = await fetch(this.slackWebhook, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(payload),
    });
    
    if (!response.ok) {
      throw new Error(`Slack webhook failed: ${response.status}`);
    }
  }

  /**
   * 发送到自定义 Webhook
   */
  async sendToWebhook(alert) {
    if (this.webhooks.length === 0) return;
    
    await Promise.all(this.webhooks.map(async (url) => {
      const response = await fetch(url, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          alert,
          timestamp: new Date().toISOString(),
          service: 'claude-sdk-app',
        }),
      });
      
      if (!response.ok) {
        throw new Error(`Webhook ${url} failed: ${response.status}`);
      }
    }));
  }

  /**
   * 发送邮件
   */
  async sendEmail(alert) {
    // 实现邮件发送逻辑（可用 nodemailer）
    // 这里仅作示意
    if (!this.emailConfig) return;
    
    logger.info({ alert, to: this.emailConfig.to }, '发送告警邮件');
  }

  /**
   * 发送到 PagerDuty
   */
  async sendToPagerDuty(alert) {
    if (!this.pagerDutyKey) return;
    
    const severity = alert.labels.severity === 'critical' ? 'critical' : 'warning';
    
    const payload = {
      routing_key: this.pagerDutyKey,
      event_action: 'trigger',
      dedup_key: `${alert.labels.alertname}-${Date.now()}`,
      payload: {
        summary: alert.annotations.summary,
        severity,
        source: 'claude-sdk-app',
        custom_details: alert.annotations,
        timestamp: new Date().toISOString(),
      },
    };
    
    const response = await fetch('https://events.pagerduty.com/v2/enqueue', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(payload),
    });
    
    if (!response.ok) {
      throw new Error(`PagerDuty API failed: ${response.status}`);
    }
  }

  _getSlackColor(severity) {
    const colors = {
      critical: 'danger',
      warning: 'warning',
      info: '#439FE0',
    };
    return colors[severity] || '#808080';
  }
}

// 使用示例
const notifier = new AlertNotifier({
  slackWebhook: process.env.SLACK_WEBHOOK_URL,
  webhooks: process.env.ALERT_WEBHOOKS?.split(',') || [],
  pagerDutyKey: process.env.PAGERDUTY_ROUTING_KEY,
});

// Prometheus Alertmanager 配置将告警推送到你的服务端点
```

## 25.6 完整示例：可观测的生产级 Claude SDK 应用

### 25.6.1 应用入口集成所有组件

```javascript
// app.js - 完整的可观测性集成示例
import express from 'express';
import { initTelemetry, shutdownTelemetry } from './telemetry.js';
import { setupMetricsEndpoint } from './prometheus-metrics.js';
import { ClaudeLogger } from './claude-logger.js';
import { ClaudeTelemetry } from './claude-telemetry.js';
import { MonitoredClaudeClient } from './monitored-client.js';
import { AlertNotifier } from './alert-notifier.js';
import logger from './logger.js';

// 初始化 OpenTelemetry
await initTelemetry();

// 初始化组件
const claudeLogger = new ClaudeLogger({ userId: 'system' });
const claudeTelemetry = new ClaudeTelemetry();
const client = new MonitoredClaudeClient({ 
  apiKey: process.env.ANTHROPIC_API_KEY 
});
const alertNotifier = new AlertNotifier({
  slackWebhook: process.env.SLACK_WEBHOOK_URL,
});

// 创建 Express 应用
const app = express();
app.use(express.json());

// 暴露 Prometheus 指标端点
setupMetricsEndpoint(app);

// 健康检查
app.get('/health', (req, res) => {
  res.json({ status: 'ok', timestamp: new Date().toISOString() });
});

// API 端点
app.post('/chat', async (req, res) => {
  const { message, userId, sessionId } = req.body;
  
  try {
    const response = await client.createMessage({
      model: 'claude-3-5-sonnet-20241022',
      max_tokens: 1024,
      messages: [{ role: 'user', content: message }],
    });
    
    res.json({
      response: response.content[0].text,
      usage: response.usage,
    });
  } catch (error) {
    logger.error({ error, userId, sessionId }, 'Chat request failed');
    res.status(500).json({ error: error.message });
  }
});

// 流式 API
app.post('/chat/stream', async (req, res) => {
  const { message, userId, sessionId } = req.body;
  
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');
  
  try {
    for await (const event of client.streamMessage({
      model: 'claude-3-5-sonnet-20241022',
      max_tokens: 1024,
      messages: [{ role: 'user', content: message }],
    })) {
      if (event.type === 'content_block_delta') {
        res.write(`data: ${JSON.stringify({ text: event.delta.text })}\n\n`);
      }
    }
    
    res.write('data: [DONE]\n\n');
    res.end();
  } catch (error) {
    logger.error({ error, userId, sessionId }, 'Stream request failed');
    res.write(`data: ${JSON.stringify({ error: error.message })}\n\n`);
    res.end();
  }
});

// 优雅关闭
const server = app.listen(process.env.PORT || 3000, () => {
  logger.info(`Server started on port ${process.env.PORT || 3000}`);
});

process.on('SIGTERM', async () => {
  logger.info('Received SIGTERM, shutting down gracefully...');
  server.close();
  await shutdownTelemetry();
  process.exit(0);
});

process.on('SIGINT', async () => {
  logger.info('Received SIGINT, shutting down gracefully...');
  server.close();
  await shutdownTelemetry();
  process.exit(0);
});
```

### 25.6.2 Docker Compose 部署（含监控栈）

```yaml
# docker-compose.yaml - 完整的可观测性部署
version: '3.8'

services:
  # 应用服务
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
      - OTEL_EXPORTER_OTLP_TRACES_ENDPOINT=http://otel-collector:4318/v1/traces
      - OTEL_EXPORTER_OTLP_METRICS_ENDPOINT=http://otel-collector:4318/v1/metrics
      - LOG_LEVEL=info
    depends_on:
      - otel-collector
      - prometheus
      - grafana

  # OpenTelemetry Collector
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    command: ["--config=/etc/otel-collector-config.yaml"]
    volumes:
      - ./otel-collector-config.yaml:/etc/otel-collector-config.yaml
    ports:
      - "4318:4318"   # OTLP HTTP receiver
      - "4317:4317"   # OTLP gRPC receiver
      - "8888:8888"   # Prometheus metrics exposed by the collector
    depends_on:
      - jaeger
      - prometheus

  # Jaeger (分布式追踪)
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"  # Jaeger UI
      - "14268:14268"  # Jaeger HTTP

  # Prometheus (指标存储)
  prometheus:
    image: prom/prometheus:latest
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
      - ./prometheus-alerts.yaml:/etc/prometheus/alerts.yaml
    ports:
      - "9090:9090"

  # Grafana (可视化)
  grafana:
    image: grafana/grafana:latest
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana-dashboards:/etc/grafana/provisioning/dashboards
    ports:
      - "3001:3000"
    depends_on:
      - prometheus
      - jaeger

  # Alertmanager (告警管理)
  alertmanager:
    image: prom/alertmanager:latest
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml
    ports:
      - "9093:9093"

volumes:
  prometheus-data:
  grafana-data:
```

### 25.6.3 OpenTelemetry Collector 配置

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      http:
        endpoint: 0.0.0.0:4318
      grpc:
        endpoint: 0.0.0.0:4317

processors:
  batch:
    timeout: 1s
    send_batch_size: 1024
  memory_limiter:
    check_interval: 1s
    limit_mib: 512

exporters:
  # 导出 traces 到 Jaeger
  jaeger:
    endpoint: jaeger:14250
    tls:
      insecure: true
  
  # 导出 metrics 到 Prometheus
  prometheus:
    endpoint: "0.0.0.0:8889"
    namespace: claude_sdk
  
  # 导出 logs (可选)
  logging:
    loglevel: info

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [jaeger]
    
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [prometheus]
    
    logs:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [logging]
```

## 25.7 最佳实践总结

### 25.7.1 日志最佳实践

| 实践 | 说明 | 示例 |
|------|------|------|
| **结构化日志** | 使用 JSON 格式，便于查询 | `{"event":"request","model":"claude-3-5-sonnet"}` |
| **统一字段名** | 团队约定字段命名规范 | `inputTokens` vs `input_tokens` |
| **日志级别** | 合理使用 debug/info/warn/error | 生产环境用 info，调试用 debug |
| **关联ID** | 每个请求分配唯一 traceId | `trace_a1b2c3d4e5f6` |
| **敏感信息** | 脱敏处理，不记录 API Key | `apiKey: "sk-***...***"` |
| **批量导出** | 避免每条日志单独写文件 | 使用 Pino 的批量模式 |

### 25.7.2 指标最佳实践

| 实践 | 说明 |
|------|------|
| **命名规范** | 使用 snake_case，带单位后缀（如 `_total`、`_seconds`） |
| **标签控制** | 避免高基数标签（如 userId、requestId），会爆炸存储 |
| **四种类型** | Counter 累计值、Gauge 瞬时值、Histogram 分布、Summary 分位数 |
| **预先规划** | 提前设计 Dashboard，再确定需要哪些指标 |

### 25.7.3 追踪最佳实践

| 实践 | 说明 |
|------|------|
| **Span 命名** | 使用动词+名词，如 `claude.api.request` |
| **属性丰富** | 记录 model、tokens、duration 等关键信息 |
| **错误记录** | 使用 `recordException` 和 `setStatus(ERROR)` |
| **采样策略** | 高流量场景可采样 1%-10% |

### 25.7.4 告警最佳实践

| 实践 | 说明 |
|------|------|
| **分级告警** | critical（立即响应）、warning（关注）、info（记录） |
| **避免告警疲劳** | 设置合理的阈值和持续时间（`for: 5m`） |
| **可操作** | 告警信息包含诊断指引 |
| **降噪** | 相关告警聚合，避免重复通知 |

## 25.8 小结

本章我们学习了如何为 Claude Code SDK 应用构建完整的可观测性体系：

1. **结构化日志** - 使用 Pino 记录结构化日志，便于查询和分析
2. **OpenTelemetry** - 统一的可观测性标准，支持 Traces、Metrics、Logs
3. **Prometheus + Grafana** - 业界主流的监控方案，丰富的可视化能力
4. **告警系统** - 主动发现问题，支持 Slack、PagerDuty、邮件等通知渠道

> 💡 **关键收获**：可观测性不是事后补丁，而是从架构设计开始就要考虑的核心能力。一个"盲飞"的生产系统是运维噩梦。

---

## 下一章预告

下一章我们将探索**附录H：社区插件与工具生态**，了解 Claude Code SDK 周边丰富的第三方工具和插件，助你事半功倍。
