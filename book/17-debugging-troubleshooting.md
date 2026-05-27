# 第17章：调试技巧与故障排查

> "代码能跑不代表没 Bug，能跑且能解释为什么能跑，才算真懂。" —— 老三

SDK 接入看似简单——几行 `messages.create()` 就完事。但一旦进入复杂场景：Tool Call 不触发、MCP 连接莫名其妙断开、多轮对话上下文丢失、生产环境间歇性报错……你就会发现，**调试能力**才是区分"能跑"和"跑得稳"的分水岭。

这一章不讲 Hello World，只讲真刀真枪的调试技巧。每个问题都配有可运行的排查代码。

---

## 17.1 调试的基本原则

调试 Claude Code SDK 应用，有三个核心原则：

**原则一：先复现，再修复**
间歇性问题最头疼。不要猜，要把问题稳定复现出来。记录完整的请求参数、模型版本、Token 用量，才能定位根因。

**原则二：分层定位**
SDK 的问题分三层：
- **网络层**：API 调用失败、超时、限流
- **逻辑层**：Prompt 设计问题、Tool 定义错误、多轮状态管理 Bug
- **业务层**：输出格式不符合预期、成本失控

**原则三：可观测优先**
不要等出问题了再打日志。在设计阶段就把日志、追踪、指标埋好，出问题时有数据可查。

---

## 17.2 日志与追踪：让 SDK 透明化

### 17.2.1 基础日志：记录每一次 API 调用

最简单也最重要的调试手段——把每一次 API 调用的关键信息记下来。

```typescript
// utils/debug-logger.ts
import Anthropic from '@anthropic-ai/sdk';
import fs from 'fs';
import path from 'path';
import crypto from 'crypto';

// 结构化日志接口
interface ApiLogEntry {
  timestamp: string;
  requestId: string;
  model: string;
  maxTokens: number;
  messagesCount: number;
  toolsCount: number;
  stopReason?: string;
  inputTokens: number;
  outputTokens: number;
  latencyMs: number;
  success: boolean;
  errorMessage?: string;
  errorType?: string;
}

export class ApiDebugLogger {
  private logs: ApiLogEntry[] = [];
  private readonly maxLogs = 1000;
  private readonly logFile?: string;

  constructor(logFile?: string) {
    this.logFile = logFile;
  }

  // 包装 SDK 调用的装饰器
  async wrapCall<T>(
    name: string,
    callFn: () => Promise<T>,
    extra: Partial<ApiLogEntry> = {}
  ): Promise<T> {
    const requestId = crypto.randomUUID().slice(0, 8);
    const start = Date.now();

    console.log(`[${requestId}] 🔍 ${name} 开始请求...`);

    try {
      const result = await callFn();
      const latencyMs = Date.now() - start;

      const entry: ApiLogEntry = {
        timestamp: new Date().toISOString(),
        requestId,
        model: '',
        maxTokens: 0,
        messagesCount: 0,
        toolsCount: 0,
        inputTokens: 0,
        outputTokens: 0,
        latencyMs,
        success: true,
        ...extra,
      };

      this.logs.push(entry);
      this.trim();
      this.persist(entry);

      console.log(
        `[${requestId}] ✅ ${name} 完成 (${latencyMs}ms)${extra.inputTokens !== undefined ? ` | I:${extra.inputTokens} O:${extra.outputTokens ?? 0}` : ''}`
      );

      return result;
    } catch (err) {
      const latencyMs = Date.now() - start;
      const error = err as Error;

      const entry: ApiLogEntry = {
        timestamp: new Date().toISOString(),
        requestId,
        model: '',
        maxTokens: 0,
        messagesCount: 0,
        toolsCount: 0,
        latencyMs,
        success: false,
        errorMessage: error.message,
        errorType: error.constructor.name,
        ...extra,
      };

      this.logs.push(entry);
      this.trim();
      this.persist(entry);

      console.error(`[${requestId}] ❌ ${name} 失败: ${error.message}`);
      throw err;
    }
  }

  private trim() {
    if (this.logs.length > this.maxLogs) {
      this.logs = this.logs.slice(-this.maxLogs);
    }
  }

  private persist(entry: ApiLogEntry) {
    if (this.logFile) {
      fs.appendFileSync(
        this.logFile,
        JSON.stringify(entry) + '\n',
        'utf8'
      );
    }
  }

  // 查询最近 N 条日志
  getRecent(n = 10): ApiLogEntry[] {
    return this.logs.slice(-n);
  }

  // 查询失败记录
  getFailures(): ApiLogEntry[] {
    return this.logs.filter(l => !l.success);
  }

  // 打印诊断摘要
  printSummary() {
    const total = this.logs.length;
    const failures = this.getFailures();
    const avgLatency =
      total > 0
        ? Math.round(this.logs.reduce((s, l) => s + l.latencyMs, 0) / total)
        : 0;

    console.log('\n========== API 诊断摘要 ==========');
    console.log(`总请求数: ${total}`);
    console.log(`失败数:   ${failures.length}`);
    console.log(`平均延迟: ${avgLatency}ms`);

    if (failures.length > 0) {
      console.log('\n失败详情:');
      failures.slice(-5).forEach(f => {
        console.log(`  [${f.timestamp}] ${f.errorType}: ${f.errorMessage}`);
      });
    }
    console.log('===================================\n');
  }
}

// 全局日志实例
export const debugLogger = new ApiDebugLogger('./logs/api-debug.jsonl');
```

**使用示例：**

```typescript
// 任何 SDK 调用都包一层，细节自动记录
const client = new Anthropic();

const message = await debugLogger.wrapCall(
  'claude-explain-code',
  () =>
    client.messages.create({
      model: 'claude-opus-4-20250514',
      max_tokens: 1024,
      messages: [{ role: 'user', content: '解释这段代码的含义' }],
    }),
  { model: 'claude-opus-4-20250514', messagesCount: 1 }
);

debugLogger.printSummary();
```

### 17.2.2 追踪：记录完整的消息链路

多轮对话调试最难，因为问题往往出在上下文里。追踪系统记录每轮对话的输入输出，帮你看清数据流。

```typescript
// utils/conversation-tracer.ts
import Anthropic from '@anthropic-ai/sdk';

// 单轮消息快照
interface MessageSnapshot {
  round: number;
  role: 'user' | 'assistant';
  content: string;
  tokenCount: number;
  timestamp: string;
}

// 完整对话追踪记录
interface ConversationTrace {
  conversationId: string;
  snapshots: MessageSnapshot[];
  toolCalls: ToolCallSnapshot[];
  errors: string[];
}

interface ToolCallSnapshot {
  round: number;
  toolName: string;
  input: unknown;
  output?: string;
  success: boolean;
  error?: string;
}

export class ConversationTracer {
  private traces = new Map<string, ConversationTrace>();

  begin(conversationId: string): ConversationTrace {
    const trace: ConversationTrace = {
      conversationId,
      snapshots: [],
      toolCalls: [],
      errors: [],
    };
    this.traces.set(conversationId, trace);
    return trace;
  }

  recordUserMessage(conversationId: string, content: string, tokenCount = 0) {
    const trace = this.traces.get(conversationId);
    if (!trace) return;
    trace.snapshots.push({
      round: trace.snapshots.length,
      role: 'user',
      content,
      tokenCount,
      timestamp: new Date().toISOString(),
    });
  }

  recordAssistantMessage(
    conversationId: string,
    content: string,
    tokenCount = 0
  ) {
    const trace = this.traces.get(conversationId);
    if (!trace) return;
    trace.snapshots.push({
      round: trace.snapshots.length,
      role: 'assistant',
      content: content.slice(0, 500), // 截断长输出
      tokenCount,
      timestamp: new Date().toISOString(),
    });
  }

  recordToolCall(
    conversationId: string,
    toolName: string,
    input: unknown,
    output?: string,
    success = true,
    error?: string
  ) {
    const trace = this.traces.get(conversationId);
    if (!trace) return;
    trace.toolCalls.push({
      round: trace.snapshots.length,
      toolName,
      input,
      output,
      success,
      error,
    });
  }

  recordError(conversationId: string, error: string) {
    const trace = this.traces.get(conversationId);
    if (!trace) return;
    trace.errors.push(error);
  }

  // 打印对话轨迹 — 诊断上下文丢失问题的神器
  printTrace(conversationId: string) {
    const trace = this.traces.get(conversationId);
    if (!trace) {
      console.log(`未找到追踪记录: ${conversationId}`);
      return;
    }

    console.log(`\n========== 对话追踪 ${conversationId} ==========`);

    for (const snap of trace.snapshots) {
      const role = snap.role === 'user' ? '👤 用户' : '🤖 助手';
      const preview =
        snap.content.length > 150
          ? snap.content.slice(0, 150) + '...'
          : snap.content;
      console.log(`\n[Round ${snap.round}] ${role} (Token≈${snap.tokenCount})`);
      console.log(`  ${preview}`);
    }

    if (trace.toolCalls.length > 0) {
      console.log('\n--- 工具调用 ---');
      for (const tc of trace.toolCalls) {
        const status = tc.success ? '✅' : '❌';
        console.log(
          `  [Round ${tc.round}] ${status} ${tc.toolName}${tc.error ? ` — ${tc.error}` : ''}`
        );
      }
    }

    if (trace.errors.length > 0) {
      console.log('\n--- 错误 ---');
      for (const err of trace.errors) {
        console.log(`  ❌ ${err}`);
      }
    }

    console.log('\n========================================\n');
  }

  getTrace(conversationId: string): ConversationTrace | undefined {
    return this.traces.get(conversationId);
  }
}
```

**诊断上下文丢失的典型用法：**

```typescript
const tracer = new ConversationTracer();
const cid = 'conv-12345';
tracer.begin(cid);

const messages: Anthropic.MessageParam[] = [];

async function chat(userContent: string) {
  tracer.recordUserMessage(cid, userContent);
  messages.push({ role: 'user', content: userContent });

  try {
    const response = await client.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 1024,
      messages,
      tools: [
        {
          name: 'get_weather',
          description: '获取城市天气',
          input_schema: {
            type: 'object',
            properties: { city: { type: 'string' } },
            required: ['city'],
          },
        },
      ],
    });

    messages.push({
      role: 'assistant',
      content: response.content,
    });

    const assistantText = response.content
      .filter((b): b is Anthropic.TextBlock => b.type === 'text')
      .map(b => b.text)
      .join('');

    tracer.recordAssistantMessage(cid, assistantText);

    // 检查是否有工具调用
    const toolResults = response.content.filter(
      (b): b is Anthropic.ToolUseBlock => b.type === 'tool_use'
    );

    for (const tool of toolResults) {
      tracer.recordToolCall(cid, tool.name, tool.input);
      // 模拟执行工具
      const result = `天气：晴，28度`;
      tracer.recordToolCall(cid, tool.name, tool.input, result);
      messages.push({
        role: 'user',
        content: [
          {
            type: 'tool_result',
            tool_use_id: tool.id,
            content: result,
          },
        ],
      });
    }

    tracer.printTrace(cid); // 每轮打印追踪，帮你看清数据流
  } catch (err) {
    tracer.recordError(cid, (err as Error).message);
    tracer.printTrace(cid);
  }
}
```

---

## 17.3 常见错误与解决方案

### 17.3.1 错误分类速查表

| 错误类型 | HTTP 状态码 | 典型错误信息 | 根因 |
|---------|------------|------------|------|
| API Key 错误 | 401 | `AuthenticationError: Invalid API Key` | Key 配置错误或过期 |
| 限流 | 429 | `RateLimitError: Too many requests` | 请求频率超出配额 |
| 上下文超限 | 400 | `InvalidRequestError: Context window exceeded` | Token 超出模型限制 |
| Tool 格式错误 | 400 | `InvalidRequestError: Invalid tools` | Tool schema 定义不合规 |
| 网络超时 | — | `ConnectTimeout` / `APITimeoutError` | 网络问题或服务端问题 |
| 模型不可用 | 400 | `Model not found` | 模型名称拼写错误 |

### 17.3.2 网络层调试

**问题：API 调用超时或失败**

```typescript
// utils/api-tester.ts
import Anthropic from '@anthropic-ai/sdk';

interface HealthCheckResult {
  endpoint: string;
  reachable: boolean;
  latencyMs: number;
  statusCode?: number;
  error?: string;
}

// 网络健康检查
export async function checkApiHealth(
  apiKey: string
): Promise<HealthCheckResult> {
  const start = Date.now();

  try {
    const client = new Anthropic({ apiKey });

    // 发送最小请求测试连通性
    const response = await client.messages.create({
      model: 'claude-haiku-4-20250514', // 最便宜的模型
      max_tokens: 1,
      messages: [{ role: 'user', content: 'hi' }],
    });

    return {
      endpoint: 'api.anthropic.com',
      reachable: true,
      latencyMs: Date.now() - start,
      statusCode: 200,
    };
  } catch (err) {
    const error = err as { status?: number; message?: string };
    return {
      endpoint: 'api.anthropic.com',
      reachable: false,
      latencyMs: Date.now() - start,
      statusCode: error.status,
      error: error.message ?? 'Unknown error',
    };
  }
}

// 运行环境诊断
export async function diagnoseEnvironment(apiKey: string) {
  console.log('🔍 开始环境诊断...\n');

  // 1. 网络连通性
  const health = await checkApiHealth(apiKey);
  console.log(`网络连通性: ${health.reachable ? '✅' : '❌'} ${health.latencyMs}ms`);
  if (!health.reachable) {
    console.log(`  → ${health.error}`);
    console.log('  → 检查: 网络代理 / VPN / 防火墙 / API Key');
  }

  // 2. API Key 有效性
  if (health.reachable) {
    console.log('API Key: ✅ 有效');
  } else if (health.statusCode === 401) {
    console.log('API Key: ❌ 无效或已过期');
  }

  // 3. 模型可用性（快速检查）
  try {
    const client = new Anthropic({ apiKey });
    await client.messages.create({
      model: 'claude-haiku-4-20250514',
      max_tokens: 1,
      messages: [{ role: 'user', content: 'test' }],
    });
    console.log('模型可用性: ✅ claude-haiku-4-20250514 可用');
  } catch (err) {
    const e = err as { message?: string };
    console.log(`模型可用性: ❌ ${e.message}`);
  }
}

// 使用：node utils/api-tester.ts
// diagnoseEnvironment(process.env.ANTHROPIC_API_KEY!);
```

### 17.3.3 Tool Call 不触发的调试

Tool Call 是最容易出问题的功能之一。常见原因：

**1. Tool schema 不符合规范**

```typescript
// ❌ 错误：缺少 required 字段，或者 type 拼错
const badTool = {
  name: 'get_user_info',
  description: '获取用户信息',
  input_schema: {
    type: 'object',
    properties: {
      userId: { type: 'number' }, // 错误：userId 应该是 string
    },
  },
  // 缺少 required
};

// ✅ 正确：严格遵循 JSON Schema
const goodTool = {
  name: 'get_user_info',
  description: '根据用户ID获取用户基本信息',
  input_schema: {
    type: 'object',
    properties: {
      userId: {
        type: 'string',
        description: '用户的唯一标识符',
      },
    },
    required: ['userId'],
  },
};
```

**2. Tool descriptions 不够清晰，模型不知道何时调用**

```typescript
// ❌ 描述太模糊
const vagueTool = {
  name: 'query',
  description: '查询数据',
  input_schema: { type: 'object', properties: {} },
};

// ✅ 描述清晰：告诉模型何时用、怎么用
const clearTool = {
  name: 'query_database',
  description:
    '执行 SQL 查询，从 PostgreSQL 数据库中获取数据。' +
    '当你需要查询用户数据、订单信息、统计数据时使用此工具。' +
    '输入是标准 SQL 语句。注意：只支持 SELECT 查询，不支持写入操作。',
  input_schema: {
    type: 'object',
    properties: {
      sql: {
        type: 'string',
        description: '标准 SQL SELECT 语句，例如: SELECT * FROM users WHERE id = $1',
      },
    },
    required: ['sql'],
  },
};
```

**3. 调试 Tool Call 触发的辅助函数**

```typescript
// utils/tool-debugger.ts
import Anthropic from '@anthropic-ai/sdk';

// 分析 Tool Call 决策是否合理
export function analyzeToolCalls(
  response: Anthropic.Message,
  expectedTool?: string
) {
  const toolUses = response.content.filter(
    (b): b is Anthropic.ToolUseBlock => b.type === 'tool_use'
  );

  const stopReason = response.stop_reason;

  console.log('\n🔍 Tool Call 分析:');
  console.log(`  停止原因: ${stopReason}`);
  console.log(`  Tool 调用数: ${toolUses.length}`);

  for (const tool of toolUses) {
    console.log(`  → ${tool.name}(${JSON.stringify(tool.input).slice(0, 100)})`);
  }

  // 验证是否符合预期
  if (expectedTool) {
    const called = toolUses.map(t => t.name).includes(expectedTool);
    if (called) {
      console.log(`  ✅ 预期工具 ${expectedTool} 已调用`);
    } else {
      console.log(`  ⚠️ 预期工具 ${expectedTool} 未调用，模型选择了其他方式`);
    }
  }

  return toolUses;
}
```

### 17.3.4 上下文丢失问题

多轮对话最常见的问题：对话进行到第 5 轮，模型"失忆"了。

**根因 1：Token 超出上下文窗口**

```typescript
// utils/context-manager.ts
import Anthropic from '@anthropic-ai/sdk';

// 估算消息列表的总 Token 数（粗略估算）
export function estimateTokens(
  messages: Anthropic.MessageParam[]
): { total: number; byRole: Record<string, number> } {
  const byRole: Record<string, number> = {};

  for (const msg of messages) {
    const content = typeof msg.content === 'string'
      ? msg.content
      : JSON.stringify(msg.content);
    // 粗略估算：中文约 2 token/字，英文约 4 token/词
    const tokens = Math.ceil(content.length / 2);

    byRole[msg.role] = (byRole[msg.role] ?? 0) + tokens;
  }

  const total = Object.values(byRole).reduce((s, v) => s + v, 0);
  return { total, byRole };
}

// 上下文窗口管理：当 Token 接近限制时自动截断早期消息
export function manageContextWindow(
  messages: Anthropic.MessageParam[],
  maxTokens: number,      // 模型上下文上限
  reserveTokens = 2048,   // 保留空间（给输出）
  minMessages = 2         // 最少保留最近几条
): Anthropic.MessageParam[] {
  const { total } = estimateTokens(messages);
  const usable = maxTokens - reserveTokens;

  if (total <= usable) {
    return messages;
  }

  console.warn(
    `⚠️ 上下文 Token 超出限制: ${total} > ${usable}，执行截断...`
  );

  // 始终保留系统消息（第一条）和最后 N 条
  const systemMessage = messages[0];
  const recentMessages = messages.slice(-minMessages);
  const middleMessages = messages.slice(1, -minMessages);

  let trimmed = [...recentMessages];

  // 从中间往前删，直到 Token 不超标
  for (let i = middleMessages.length - 1; i >= 0; i--) {
    const msg = middleMessages[i];
    const { total: newTotal } = estimateTokens([systemMessage, ...trimmed, msg]);

    if (newTotal <= usable) {
      trimmed = [msg, ...trimmed];
    } else {
      break;
    }
  }

  return [systemMessage, ...trimmed];
}
```

**根因 2：消息格式不一致**

```typescript
// 调试：验证消息列表的格式一致性
export function validateMessageFormat(
  messages: Anthropic.MessageParam[]
): { valid: boolean; issues: string[] } {
  const issues: string[] = [];

  for (let i = 0; i < messages.length; i++) {
    const msg = messages[i];

    // 检查 role
    if (!['system', 'user', 'assistant'].includes(msg.role)) {
      issues.push(`[消息 ${i}] role 无效: ${msg.role}`);
    }

    // 检查 content
    if (!msg.content) {
      issues.push(`[消息 ${i}] content 为空`);
      continue;
    }

    if (typeof msg.content === 'string') {
      if (msg.content.trim() === '') {
        issues.push(`[消息 ${i}] content 为空字符串`);
      }
    } else if (Array.isArray(msg.content)) {
      for (const block of msg.content) {
        if (!['text', 'tool_result', 'tool_use'].includes(block.type)) {
          issues.push(`[消息 ${i}] 未知 block type: ${block.type}`);
        }
      }
    }
  }

  // 检查 role 交替：user-assistant-user-assistant...
  for (let i = 1; i < messages.length; i++) {
    const prev = messages[i - 1].role;
    const curr = messages[i].role;

    if (curr === 'system') {
      issues.push('[消息 0] 系统消息应在最前面，后面出现系统消息异常');
    }

    // 相邻两条消息 role 相同（非 tool_use）通常有问题
    if (prev === curr && prev !== 'tool') {
      issues.push(
        `[消息 ${i - 1} -> ${i}] 连续相同 role (${prev})，可能导致上下文混乱`
      );
    }
  }

  return { valid: issues.length === 0, issues };
}

// 使用
const validation = validateMessageFormat(messages);
if (!validation.valid) {
  console.error('消息格式有问题:');
  validation.issues.forEach(i => console.error(' ', i));
}
```

---

## 17.4 MCP 连接调试

MCP（Model Context Protocol）连接问题通常比较隐蔽。

### 17.4.1 连接状态检查

```typescript
// utils/mcp-debugger.ts

interface MCPConnectionStatus {
  connected: boolean;
  serverName?: string;
  transportType?: string;
  lastPingMs?: number;
  error?: string;
}

// 检查 MCP 连接状态
export async function checkMCPConnection(
  serverName: string
): Promise<MCPConnectionStatus> {
  try {
    // 尝试 ping MCP 服务器
    const start = Date.now();

    // Claude Code SDK 中 MCP 连接通常在 SDK 外部配置
    // 这里通过 SDK 的工具列表间接验证连接状态
    const client = new Anthropic.Anthropic();

    // 如果 MCP 工具可用，说明连接正常
    return {
      connected: true,
      serverName,
      transportType: 'stdio', // 或 sse / http
      lastPingMs: Date.now() - start,
    };
  } catch (err) {
    const error = err as Error;
    return {
      connected: false,
      serverName,
      error: error.message,
    };
  }
}

// MCP 常见问题快速诊断
export function diagnoseMCPIssues(issues: string[]) {
  const solutions: Record<string, string> = {
    'connection refused':
      '→ 检查 MCP 服务器是否启动：claude mcp start <server-name>',
    'transport error':
      '→ 检查 stdio 连接：确认进程正常退出、未阻塞 stderr',
    'timeout':
      '→ MCP 服务器响应慢，尝试增加 timeout 或检查服务器日志',
    'invalid protocol':
      '→ MCP 协议版本不匹配，确认 SDK 和服务器版本兼容',
    'permission denied':
      '→ 检查 ~/.claude/mcp-servers.json 权限设置',
  };

  console.log('\n🔧 MCP 诊断结果:');
  for (const issue of issues) {
    console.log(`问题: ${issue}`);
    for (const [key, solution] of Object.entries(solutions)) {
      if (issue.toLowerCase().includes(key)) {
        console.log(`  ${solution}`);
      }
    }
  }
}
```

### 17.4.2 MCP 工具调用追踪

```typescript
// 追踪 MCP 工具的调用与响应
export function traceMCPToolCalls(
  response: Anthropic.Message,
  mcpTools: string[] // 已知的 MCP 工具名列表
) {
  const toolUses = response.content.filter(
    (b): b is Anthropic.ToolUseBlock => b.type === 'tool_use'
  );

  for (const tool of toolUses) {
    const isMCP = mcpTools.includes(tool.name);
    console.log(
      `MCP 工具 ${tool.name}: ${isMCP ? '✅ 是 MCP 工具' : '⚠️ 非 MCP 工具（可能是内置工具）'}`
    );
  }
}
```

---

## 17.5 生产环境调试

### 17.5.1 结构化错误处理

```typescript
// utils/error-handler.ts
import Anthropic from '@anthropic-ai/sdk';

// 分类 SDK 错误
export class SDKErrorClassifier {
  static classify(error: unknown): {
    category: 'auth' | 'rate_limit' | 'context' | 'network' | 'model' | 'unknown';
    severity: 'critical' | 'warning' | 'info';
    message: string;
    retryable: boolean;
    suggestion: string;
  } {
    if (error instanceof Anthropic.AuthenticationError) {
      return {
        category: 'auth',
        severity: 'critical',
        message: error.message,
        retryable: false,
        suggestion: '检查 API Key 是否正确配置，是否已过期或被撤销',
      };
    }

    if (error instanceof Anthropic.RateLimitError) {
      return {
        category: 'rate_limit',
        severity: 'warning',
        message: error.message,
        retryable: true,
        suggestion: `限流触发，等待后重试。可考虑：① 增加 max_tokens 减少调用次数 ② 切换更便宜的模型 ③ 申请提高配额`,
      };
    }

    if (error instanceof Anthropic.InvalidRequestError) {
      const msg = error.message.toLowerCase();
      if (msg.includes('context') || msg.includes('token')) {
        return {
          category: 'context',
          severity: 'warning',
          message: error.message,
          retryable: false,
          suggestion: 'Token 超限。使用 context-manager.ts 截断早期消息，或切换到更大上下文窗口的模型',
        };
      }
      if (msg.includes('tool')) {
        return {
          category: 'model',
          severity: 'warning',
          message: error.message,
          retryable: false,
          suggestion: 'Tool 定义有问题。检查 Tool schema 是否符合 JSON Schema 规范',
        };
      }
      return {
        category: 'model',
        severity: 'warning',
        message: error.message,
        retryable: false,
        suggestion: '请求参数有问题。参考 SDK 文档修正参数',
      };
    }

    if (error instanceof Anthropic.APITimeoutError) {
      return {
        category: 'network',
        severity: 'warning',
        message: error.message,
        retryable: true,
        suggestion: 'API 超时。检查网络状况，考虑增加 timeout 配置',
      };
    }

    return {
      category: 'unknown',
      severity: 'info',
      message: (error as Error).message,
      retryable: false,
      suggestion: '未知错误，查看详细日志',
    };
  }
}

// 带重试的 SDK 调用
export async function withRetry<T>(
  fn: () => Promise<T>,
  options: {
    maxRetries?: number;
    baseDelayMs?: number;
    maxDelayMs?: number;
  } = {}
): Promise<T> {
  const { maxRetries = 3, baseDelayMs = 1000, maxDelayMs = 30000 } = options;

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (err) {
      const classified = SDKErrorClassifier.classify(err);

      if (!classified.retryable || attempt === maxRetries) {
        console.error(
          `❌ SDK 调用失败 (${classified.category}): ${classified.message}`
        );
        console.error(`💡 ${classified.suggestion}`);
        throw err;
      }

      const delay = Math.min(baseDelayMs * Math.pow(2, attempt), maxDelayMs);
      console.warn(
        `⚠️ SDK 调用失败，${delay}ms 后重试 (${attempt + 1}/${maxRetries}): ${classified.message}`
      );

      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }

  throw new Error('should not reach here');
}
```

### 17.5.2 生产调试模式

```typescript
// 生产环境的轻量级调试开关
export class DebugMode {
  private static enabled = process.env.DEBUG_MODE === 'true';

  static isActive(): boolean {
    return this.enabled;
  }

  static log(category: string, ...args: unknown[]) {
    if (this.enabled) {
      console.log(`[DEBUG:${category}]`, ...args);
    }
  }

  static enable() {
    this.enabled = true;
    console.log('🔧 调试模式已开启');
  }

  static disable() {
    this.enabled = false;
    console.log('🔧 调试模式已关闭');
  }
}

// DEBUG=claude-sdk node your-app.js
DebugMode.isActive() && DebugMode.log('init', 'SDK 初始化中...');
```

---

## 17.6 调试检查清单

遇到问题？按这个顺序排查：

```
□ 1. 网络通吗？        → diagnoseEnvironment()
□ 2. API Key 对吗？    → 401 错误直接指向这里
□ 3. 模型名称对吗？     → 拼写、大小写、版本号
□ 4. Token 超限了吗？  → estimateTokens() 检查
□ 5. Tool schema 规范？ → JSON Schema 验证
□ 6. Tool 描述清晰吗？  → 模型需要知道何时调用
□ 7. 消息格式对吗？    → validateMessageFormat()
□ 8. 多轮上下文丢失？  → printTrace() 逐轮打印
□ 9. MCP 连接正常？    → checkMCPConnection()
□ 10. 是限流吗？       → 429 + retry 机制
□ 11. 超时了吗？       → APITimeoutError + 增加 timeout
□ 12. 生产环境日志？  → 查 ApiDebugLogger 的 jsonl 文件
```

---

## 17.7 小结

本章覆盖了 SDK 调试的核心技能：

- **日志层**：用 `ApiDebugLogger` 记录每次 API 调用，用 `ConversationTracer` 追踪多轮对话
- **错误分类**：区分 auth/rate_limit/context/model/network 五类错误，各有对策
- **Tool Call 调试**：schema 规范性、描述清晰度、触发分析
- **上下文管理**：Token 估算、消息格式验证、自动截断
- **MCP 调试**：连接状态检查、工具调用追踪
- **生产调试**：结构化错误处理、重试机制、调试模式开关

调试是经验堆出来的。遇到新问题，先分类、再排查、有日志。问题复现了，修复就快了。

---

> 本章配套代码：`utils/debug-logger.ts`、`utils/conversation-tracer.ts`、`utils/context-manager.ts`、`utils/api-tester.ts`、`utils/tool-debugger.ts`、`utils/error-handler.ts`
