# 第20章：测试策略与 Mock 技巧

> "没写测试的代码，就像没系安全带的杂技演员——可能没事，但出事就是大事。" —— 老三

Claude Code SDK 的调用依赖外部 API，没法像普通函数那样直接 assert。测试怎么写？Mock 怎么做？集成测试怎么搭？这一章统统搞定。

你将学到：
- 三层测试金字塔：单元测试、集成测试、端到端测试
- Mock API 响应的多种策略
- 完整的 Mock Server 实现
- Jest / Vitest 测试套件实战
- 覆盖 Tool Call、Streaming 等特殊场景的测试技巧

---

## 20.1 测试策略总览

在写代码之前，先想清楚测什么、怎么测。

### 20.1.1 测试金字塔

Claude Code SDK 应用有三层测试：

| 层级 | 覆盖范围 | 运行频率 | 速度 | Mock 程度 |
|------|---------|---------|------|----------|
| **单元测试** | 业务逻辑、Prompt 模板、工具函数 | 每次 commit | ⚡ <100ms | 全部 Mock |
| **集成测试** | SDK 调用链路、Tool 定义解析 | 每次 PR | 🐢 1-10s | Mock API（部分真实） |
| **端到端测试** | 完整对话流程、CLI 集成 | 每日 / 上线前 | 🐌 >10s | 零 Mock，真实 API |

**实战建议**：投入 70% 精力在单元测试，因为这里覆盖的是你的核心业务逻辑；集成测试覆盖 SDK 调用链路；E2E 只测关键路径。

### 20.1.2 测试目录结构

```
src/
  __tests__/
    unit/           ← 单元测试
      prompt.test.ts
      tool-parser.test.ts
      token-counter.test.ts
    integration/    ← 集成测试
      sdk-client.test.ts
      tool-call.test.ts
    mocks/          ← Mock 数据
      api-responses.ts
      mock-server.ts
```

---

## 20.2 Mock API 响应：四种策略

这是测试 SDK 应用的核心技能——**把 Anthropic API 挡在门外，用假数据喂你的代码**。

### 20.2.1 策略一：直接 Mock SDK 方法

最简单粗暴的方式，直接 monkey-patch 掉 SDK 的方法。

```typescript
// __tests__/mocks/sdk-mock.ts
import Anthropic from '@anthropic-ai/sdk';

// 定义 Mock 响应数据结构
export interface MockMessage {
  id: string;
  type: 'message';
  role: 'assistant';
  content: Array<{ type: 'text'; text: string } | { type: 'tool_use'; id: string; name: string; input: Record<string, unknown> }>;
  model: string;
  stop_reason: string;
  stop_sequence: null | string;
  usage: { input_tokens: number; output_tokens: number };
}

// 预设的 Mock 响应
const mockResponses: Record<string, MockMessage> = {
  hello: {
    id: 'msg_mock_001',
    type: 'message',
    role: 'assistant',
    content: [{ type: 'text', text: '你好！有什么可以帮你的？' }],
    model: 'claude-sonnet-4-20250514',
    stop_reason: 'end_turn',
    stop_sequence: null,
    usage: { input_tokens: 10, output_tokens: 20 },
  },
  tool_call: {
    id: 'msg_mock_002',
    type: 'message',
    role: 'assistant',
    content: [{
      type: 'tool_use',
      id: 'toolu_mock_001',
      name: 'calculator',
      input: { expression: '2 + 2' },
    }],
    model: 'claude-sonnet-4-20250514',
    stop_reason: 'tool_use',
    stop_sequence: null,
    usage: { input_tokens: 50, output_tokens: 30 },
  },
};

/**
 * 创建 Mock Client 的工厂函数
 */
export function createMockClient(overrides?: Partial<MockMessage>) {
  const client = new Anthropic();

  // Mock messages.create 方法
  jest.spyOn(client.messages, 'create').mockImplementation(
    async (params: { system?: string; messages: Array<{ role: string; content: string | Array<unknown> }> }) => {
      // 根据消息内容选择 Mock 响应
      const lastMessage = params.messages[params.messages.length - 1];
      const content = typeof lastMessage?.content === 'string'
        ? lastMessage.content
        : (lastMessage?.content as Array<{ type: string; text?: string }>)?.find(c => c.type === 'text')?.text || '';

      const key = content.includes('工具') || content.includes('tool') ? 'tool_call' : 'hello';

      return {
        ...mockResponses[key],
        ...overrides,
      };
    }
  );

  // Mock messages.stream 方法（见 20.5 节流式测试）
  // ...

  return client;
}
```

使用方式：

```typescript
// __tests__/integration/sdk-client.test.ts
import { createMockClient } from '../mocks/sdk-mock';
import { MyAssistant } from '../../src/my-assistant';

describe('MyAssistant', () => {
  it('应该正确处理用户问候语', async () => {
    const mockClient = createMockClient();
    const assistant = new MyAssistant(mockClient);

    const response = await assistant.chat('你好！');
    expect(response).toContain('你好');
    // ✅ 不需要真实 API key，不需要联网
  });

  it('应该在需要工具时触发 Tool Call', async () => {
    const mockClient = createMockClient();
    const assistant = new MyAssistant(mockClient);

    const response = await assistant.chat('帮我计算 2+2');
    expect(mockClient.messages.create).toHaveBeenCalledWith(
      expect.objectContaining({
        tools: expect.any(Array),
      })
    );
  });
});
```

### 20.2.2 策略二：Mock Server（更真实的模拟）

Mock SDK 方法虽然简单，但只能模拟"成功返回"。现实中的测试需要模拟各种场景：网络超时、429 限流、500 错误……用 **Mock HTTP Server** 来模拟真实的 API 行为。

```typescript
// __tests__/mocks/mock-server.ts
import http from 'http';
import { Agent } from 'http';

/**
 * 创建 Mock Anthropic API Server
 * 监听在本地端口，模拟真实的 API 响应和错误
 */
export class MockAnthropicServer {
  private server: http.Server;
  private baseUrl: string;
  private responses: Map<string, MockResponse>;
  private requestLog: Array<{ path: string; method: string; body: unknown; timestamp: number }> = [];

  constructor(private port = 3141) {
    this.baseUrl = `http://localhost:${this.port}`;
    this.server = http.createServer(this.handleRequest.bind(this));
    this.responses = new Map();
    this.setupDefaultResponses();
  }

  private setupDefaultResponses() {
    // 默认成功响应
    this.responses.set('/v1/messages', {
      status: 200,
      body: {
        id: 'msg_test_001',
        type: 'message',
        role: 'assistant',
        content: [{ type: 'text', text: 'Mock 响应成功' }],
        model: 'claude-sonnet-4-20250514',
        stop_reason: 'end_turn',
        stop_sequence: null,
        usage: { input_tokens: 10, output_tokens: 15 },
      },
    });

    // 限流响应
    this.responses.set('/v1/messages:429', {
      status: 429,
      body: { type: 'error', error: { type: 'rate_limit_error', message: 'Rate limit exceeded' } },
    });

    // 服务器错误
    this.responses.set('/v1/messages:500', {
      status: 500,
      body: { type: 'error', error: { type: 'api_error', message: 'Internal server error' } },
    });
  }

  private handleRequest(req: http.IncomingMessage, res: http.ServerResponse) {
    const url = req.url || '';
    const method = req.method || 'GET';

    // 读取请求体
    let body = '';
    req.on('data', chunk => { body += chunk; });
    req.on('end', () => {
      try {
        const parsedBody = body ? JSON.parse(body) : {};
        this.requestLog.push({ path: url, method, body: parsedBody, timestamp: Date.now() });

        // 查找匹配的 Mock 响应（支持自定义覆盖）
        const mock = this.responses.get(url) || this.responses.get('/v1/messages');
        if (!mock) {
          res.writeHead(404);
          res.end(JSON.stringify({ error: 'No mock defined' }));
          return;
        }

        res.writeHead(mock.status, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify(mock.body));
      } catch (e) {
        res.writeHead(500);
        res.end(JSON.stringify({ error: 'Mock server error' }));
      }
    });
  }

  /** 启动服务器 */
  start(): Promise<void> {
    return new Promise(resolve => this.server.listen(this.port, resolve));
  }

  /** 停止服务器 */
  stop(): Promise<void> {
    return new Promise((resolve, reject) => {
      this.server.close(err => err ? reject(err) : resolve());
    });
  }

  /** 覆盖指定端点的响应 */
  override(path: string, response: MockResponse) {
    this.responses.set(path, response);
  }

  /** 获取请求日志（用于断言） */
  getRequestLog() {
    return [...this.requestLog];
  }

  /** 获取可作为 SDK baseURL 的地址 */
  getUrl() {
    return this.baseUrl;
  }
}

interface MockResponse {
  status: number;
  body: unknown;
}
```

使用 Mock Server 配合真实 SDK：

```typescript
// __tests__/integration/error-handling.test.ts
import Anthropic from '@anthropic-ai/sdk';
import { MockAnthropicServer } from '../mocks/mock-server';

describe('SDK 错误处理集成测试', () => {
  let mockServer: MockAnthropicServer;
  let client: Anthropic;

  beforeAll(async () => {
    mockServer = new MockAnthropicServer(3141);
    await mockServer.start();

    client = new Anthropic({
      apiKey: 'test-key', // Mock server 不校验 key
      baseURL: mockServer.getUrl(), // 指向本地 Mock Server
    });
  });

  afterAll(async () => {
    await mockServer.stop();
  });

  it('应该在 429 时正确处理限流', async () => {
    mockServer.override('/v1/messages', {
      status: 429,
      body: { type: 'error', error: { type: 'rate_limit_error', message: 'Rate limit exceeded' } },
    });

    await expect(client.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 1024,
      messages: [{ role: 'user', content: 'Hello' }],
    })).rejects.toThrow('rate_limit_error');
  });

  it('应该在 500 时正确处理服务端错误', async () => {
    mockServer.override('/v1/messages', {
      status: 500,
      body: { type: 'error', error: { type: 'api_error', message: 'Internal server error' } },
    });

    await expect(client.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 1024,
      messages: [{ role: 'user', content: 'Hello' }],
    })).rejects.toThrow();
  });

  it('应该记录所有发送的请求', async () => {
    mockServer.override('/v1/messages', {
      status: 200,
      body: {
        id: 'msg_001', type: 'message', role: 'assistant',
        content: [{ type: 'text', text: 'Hi there!' }],
        model: 'claude-sonnet-4-20250514',
        stop_reason: 'end_turn', stop_sequence: null,
        usage: { input_tokens: 5, output_tokens: 10 },
      },
    });

    await client.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 1024,
      messages: [{ role: 'user', content: 'Say hi!' }],
    });

    const log = mockServer.getRequestLog();
    expect(log).toHaveLength(1);
    expect(log[0].body.messages[0].content).toBe('Say hi!');
  });
});
```

### 20.2.3 策略三：msw（Mock Service Worker）— 浏览器环境首选

如果你的应用是前端或需要在浏览器中测试，**MSW** 是最专业的选择。它拦截浏览器的 HTTP 请求，不需要改任何代码。

```typescript
// __tests__/browser/llm-client.test.ts
import { setupServer } from 'msw/node';
import { http, HttpResponse } from 'msw';

// 定义 Mock handlers
const server = setupServer(
  http.post('https://api.anthropic.com/v1/messages', () => {
    return HttpResponse.json({
      id: 'msg_browser_001',
      type: 'message',
      role: 'assistant',
      content: [{ type: 'text', text: '浏览器环境 Mock 成功！' }],
      model: 'claude-sonnet-4-20250514',
      stop_reason: 'end_turn',
      stop_sequence: null,
      usage: { input_tokens: 8, output_tokens: 25 },
    });
  })
);

beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));
afterAll(() => server.close());
afterEach(() => server.resetHandlers());

describe('浏览器端 SDK 测试', () => {
  it('MSW 可以 Mock 浏览器中的 SDK 请求', async () => {
    const client = new Anthropic();
    const message = await client.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 1024,
      messages: [{ role: 'user', content: 'test' }],
    });
    expect(message.content[0].type).toBe('text');
    expect((message.content[0] as { text: string }).text).toBe('浏览器环境 Mock 成功！');
  });
});
```

### 20.2.4 策略四：VCR / Recording（录制回放）

这是最懒但最真实的策略：**第一次跑测试时连真实 API，录下来；之后跑测试时播放录音**。

```typescript
// __tests__/utils/vcr-recorder.ts
import fs from 'fs';
import path from 'path';
import crypto from 'crypto';

interface RecordedInteraction {
  request: {
    url: string;
    method: string;
    headers: Record<string, string>;
    body: unknown;
  };
  response: {
    status: number;
    body: unknown;
  };
  recordedAt: string;
}

/**
 * VCR 风格的录制回放工具
 * 第一次运行：发真实请求，录制响应
 * 后续运行：直接读录制文件，不发网络请求
 */
export class VCRRecorder {
  constructor(private recordingsDir: string) {
    if (!fs.existsSync(recordingsDir)) {
      fs.mkdirSync(recordingsDir, { recursive: true });
    }
  }

  private getRecordingPath(params: unknown): string {
    const hash = crypto.createHash('md5').update(JSON.stringify(params)).digest('hex');
    return path.join(this.recordingsDir, `${hash}.json`);
  }

  private generateKey(params: unknown): string {
    return crypto.createHash('md5').update(JSON.stringify(params)).digest('hex');
  }

  /**
   * 录制或回放
   * @param params 请求参数
   * @param fetcher 真实的 fetch 函数
   * @param recordMode 是否强制录制（跳过已有录音）
   */
  async playOrRecord<T>(
    params: unknown,
    fetcher: () => Promise<T>,
    recordMode = false
  ): Promise<T> {
    const key = this.generateKey(params);
    const filePath = this.getRecordingPath(params);

    // 回放模式：有录音文件，直接返回
    if (!recordMode && fs.existsSync(filePath)) {
      const recorded = JSON.parse(fs.readFileSync(filePath, 'utf-8')) as RecordedInteraction;
      console.log(`[VCR] Replaying: ${key}`);
      return recorded.response.body as T;
    }

    // 录制模式：发真实请求并保存
    console.log(`[VCR] Recording: ${key}`);
    const response = await fetcher();

    const interaction: RecordedInteraction = {
      request: { url: '', method: 'POST', headers: {}, body: params },
      response: { status: 200, body: response },
      recordedAt: new Date().toISOString(),
    };

    fs.writeFileSync(filePath, JSON.stringify(interaction, null, 2));
    return response;
  }

  /** 清空所有录音 */
  clearAll() {
    const files = fs.readdirSync(this.recordingsDir);
    files.forEach(f => fs.unlinkSync(path.join(this.recordingsDir, f)));
  }
}
```

使用示例：

```typescript
// __tests__/vcr/real-api.test.ts
import { VCRRecorder } from '../utils/vcr-recorder';
import Anthropic from '@anthropic-ai/sdk';

const vcr = new VCRRecorder('./__tests__/recordings');

describe('真实 API 录制回放', () => {
  it('首次运行录制，之后运行回放', async () => {
    const client = new Anthropic();
    const params = {
      model: 'claude-sonnet-4-20250514',
      max_tokens: 100,
      messages: [{ role: 'user', content: '1+1等于几？' }],
    };

    const response = await vcr.playOrRecord(
      params,
      () => client.messages.create(params as Parameters<typeof client.messages.create>[0])
    );

    expect(response).toHaveProperty('content');
  });
});
```

---

## 20.3 单元测试：Jest / Vitest 实战

### 20.3.1 Prompt 模板测试

Prompt 是 SDK 应用的核心，**Prompt 模板必须有测试**。

```typescript
// src/utils/prompt-builder.ts
export function buildSystemPrompt(context: {
  userName: string;
  expertise: string[];
  tone: 'formal' | 'casual';
}): string {
  const expertiseList = context.expertise.map(e => `  - ${e}`).join('\n');
  const toneLine = context.tone === 'formal'
    ? '请使用正式、专业的语言风格。'
    : '请使用轻松、口语化的风格。';

  return `你是一位专业助手，用户信息如下：
用户名：${context.userName}
专业领域：
${expertiseList}

${toneLine}

记住：始终基于用户的专业领域来调整你的解释深度。`;
}
```

```typescript
// __tests__/unit/prompt-builder.test.ts
import { describe, it, expect } from 'vitest'; // 或 from '@jest/globals' + jest
import { buildSystemPrompt } from '../../src/utils/prompt-builder';

describe('Prompt 模板测试', () => {
  it('应该正确注入用户名', () => {
    const prompt = buildSystemPrompt({
      userName: '张三',
      expertise: ['Python', 'React'],
      tone: 'casual',
    });
    expect(prompt).toContain('张三');
    expect(prompt).toContain('Python');
    expect(prompt).toContain('React');
  });

  it('应该根据 tone 选择正式语气', () => {
    const formal = buildSystemPrompt({
      userName: '李四', expertise: ['法律'], tone: 'formal',
    });
    expect(formal).toContain('正式');

    const casual = buildSystemPrompt({
      userName: '王五', expertise: ['游戏'], tone: 'casual',
    });
    expect(casual).toContain('口语化');
  });

  it('应该处理空专业领域', () => {
    const prompt = buildSystemPrompt({
      userName: '测试', expertise: [], tone: 'casual',
    });
    expect(prompt).toContain('测试');
    expect(prompt).toContain('专业领域');
  });
});
```

### 20.3.2 Tool 定义解析测试

Tool 是 SDK 的核心能力，Tool 定义解析必须有测试覆盖。

```typescript
// src/utils/tool-parser.ts
export interface ParsedTool {
  name: string;
  description: string;
  requiredParams: string[];
  optionalParams: string[];
}

export function parseToolDefinition(tool: { name: string; description: string; input_schema: Record<string, unknown> }): ParsedTool {
  const requiredParams: string[] = [];
  const optionalParams: string[] = [];
  const schema = tool.input_schema as { required?: string[]; properties?: Record<string, unknown> };

  if (schema.required) {
    requiredParams.push(...schema.required);
  }
  if (schema.properties) {
    Object.keys(schema.properties).forEach(key => {
      if (!schema.required?.includes(key)) {
        optionalParams.push(key);
      }
    });
  }

  return {
    name: tool.name,
    description: tool.description,
    requiredParams,
    optionalParams,
  };
}
```

```typescript
// __tests__/unit/tool-parser.test.ts
import { describe, it, expect } from 'vitest';
import { parseToolDefinition } from '../../src/utils/tool-parser';

describe('Tool 定义解析', () => {
  it('应该正确分离必填和可选参数', () => {
    const tool = {
      name: 'calculator',
      description: '执行数学计算',
      input_schema: {
        required: ['expression'],
        properties: {
          expression: { type: 'string', description: '数学表达式' },
          precision: { type: 'integer', description: '小数位数' },
        },
      },
    };

    const parsed = parseToolDefinition(tool);
    expect(parsed.name).toBe('calculator');
    expect(parsed.requiredParams).toEqual(['expression']);
    expect(parsed.optionalParams).toEqual(['precision']);
  });

  it('应该处理没有可选参数的情况', () => {
    const tool = {
      name: 'ping',
      description: '检查服务可用性',
      input_schema: {
        required: [],
        properties: {},
      },
    };

    const parsed = parseToolDefinition(tool);
    expect(parsed.requiredParams).toHaveLength(0);
    expect(parsed.optionalParams).toHaveLength(0);
  });

  it('应该处理没有 input_schema 的情况', () => {
    const tool = {
      name: 'no-params',
      description: '无需参数的工具',
      input_schema: {},
    };

    const parsed = parseToolDefinition(tool);
    expect(parsed.requiredParams).toHaveLength(0);
    expect(parsed.optionalParams).toHaveLength(0);
  });
});
```

---

## 20.4 集成测试：覆盖完整对话流程

集成测试验证 SDK 调用链路，需要测试完整的用户请求 → SDK 调用 → 响应处理流程。

```typescript
// src/smart-assistant.ts — 一个小助手类，用于集成测试
import Anthropic from '@anthropic-ai/sdk';

export class SmartAssistant {
  constructor(private client: Anthropic) {}

  async chat(message: string) {
    const response = await this.client.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 1024,
      messages: [{ role: 'user', content: message }],
    });

    const content = response.content[0];
    if (content.type === 'text') {
      return { type: 'text' as const, text: content.text };
    }
    if (content.type === 'tool_use') {
      return {
        type: 'tool_call' as const,
        toolName: content.name,
        input: content.input,
      };
    }
    throw new Error(`Unknown content type: ${content.type}`);
  }
}
```

```typescript
// __tests__/integration/smart-assistant.test.ts
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import { SmartAssistant } from '../../src/smart-assistant';
import { createMockClient } from '../mocks/sdk-mock';

describe('SmartAssistant 集成测试', () => {
  let assistant: SmartAssistant;

  describe('正常对话流程', () => {
    beforeEach(() => {
      assistant = new SmartAssistant(createMockClient());
    });

    it('应该返回文本回复', async () => {
      const result = await assistant.chat('你好');
      expect(result.type).toBe('text');
    });

    it('应该正确处理 Tool Call', async () => {
      const result = await assistant.chat('帮我调用计算器');
      expect(result.type).toBe('tool_call');
    });
  });

  describe('错误处理', () => {
    it('应该抛出非文本/工具类型的错误', async () => {
      // 创建返回非预期 content type 的 Mock
      const badClient = createMockClient({
        content: [{ type: 'invalid_type', data: 'test' }] as unknown as SmartAssistant extends { chat(msg: string): infer R } ? R extends Promise<infer T> ? T extends { type: 'text' | 'tool_call' } ? never : unknown : unknown : never,
      });
      assistant = new SmartAssistant(badClient);

      await expect(assistant.chat('trigger error')).rejects.toThrow('Unknown content type');
    });
  });
});
```

---

## 20.5 流式响应测试

Streaming 是 SDK 的高级特性，测试流式响应需要特殊的 Mock 策略。

```typescript
// __tests__/unit/streaming.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest';

describe('流式响应测试', () => {
  beforeEach(() => {
    vi.clearAllMocks();
  });

  it('应该正确解析 SSE 事件流', () => {
    // 模拟 SSE 响应字符串（来自 Anthropic streaming API）
    const mockSSEData = [
      'event: message_start\ndata: {"type":"message_start","message":{"id":"msg_001","type":"message","role":"assistant","content":[]}}\n\n',
      'event: content_block_start\ndata: {"type":"content_block_start","index":0,"content_block":{"type":"text","text":""}}\n\n',
      'event: content_block_delta\ndata: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"你好"}}\n\n',
      'event: content_block_delta\ndata: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"世界"}}\n\n',
      'event: message_stop\ndata: {"type":"message_stop"}\n\n',
    ].join('');

    // 解析 SSE 事件的工具函数
    function parseSSEEvents(data: string) {
      const events: Array<{ event: string; data: unknown }> = [];
      const lines = data.split('\n');
      let currentEvent = '';
      let currentData = '';

      for (const line of lines) {
        if (line.startsWith('event: ')) {
          currentEvent = line.slice(7);
        } else if (line.startsWith('data: ')) {
          currentData = line.slice(6);
        } else if (line === '') {
          if (currentEvent && currentData) {
            events.push({
              event: currentEvent,
              data: JSON.parse(currentData),
            });
            currentEvent = '';
            currentData = '';
          }
        }
      }
      return events;
    }

    const events = parseSSEEvents(mockSSEData);
    expect(events).toHaveLength(5);
    expect(events[2].data).toEqual({ type: 'text_delta', text: '你好' });
    expect(events[3].data).toEqual({ type: 'text_delta', text: '世界' });

    // 验证文本累积
    const textDeltas = events
      .filter(e => e.event === 'content_block_delta')
      .map(e => (e.data as { delta: { text: string } }).delta.text);
    expect(textDeltas.join('')).toBe('你好世界');
  });

  it('应该正确处理流式 Tool Call', () => {
    const toolUseSSE = [
      'event: content_block_start\ndata: {"type":"content_block_start","index":0,"content_block":{"type":"tool_use","name":"","input":{}}}\n\n',
      'event: content_block_delta\ndata: {"type":"content_block_delta","index":0,"delta":{"type":"input_json_delta","partial_json":"{\\"expression\\": \\"2+2\\""}}\n\n',
      'event: content_block_delta\ndata: {"type":"content_block_delta","index":0,"delta":{"type":"input_json_delta","partial_json":"}"}}\n\n',
      'event: content_block_stop\ndata: {"type":"content_block_stop"}\n\n',
    ].join('');

    function parseToolUseFromSSE(data: string): { name: string; input: Record<string, unknown> } | null {
      let result: { name: string; input: Record<string, unknown> } | null = null;
      const lines = data.split('\n');
      let currentBlock: Record<string, unknown> = {};
      let partialJson = '';

      for (const line of lines) {
        if (line.startsWith('data: ')) {
          const eventData = JSON.parse(line.slice(6));
          if (eventData.type === 'content_block_start' && eventData.content_block?.type === 'tool_use') {
            currentBlock = { name: eventData.content_block.name, input: {} };
          }
          if (eventData.type === 'content_block_delta' && eventData.delta?.type === 'input_json_delta') {
            partialJson += eventData.delta.partial_json;
          }
          if (eventData.type === 'content_block_stop' && Object.keys(currentBlock).length > 0) {
            try {
              currentBlock.input = JSON.parse(partialJson);
              result = currentBlock as { name: string; input: Record<string, unknown> };
            } catch {
              // JSON 尚未完成，忽略
            }
          }
        }
      }
      return result;
    }

    const toolUse = parseToolUseFromSSE(toolUseSSE);
    expect(toolUse).not.toBeNull();
    expect(toolUse?.input).toEqual({ expression: '2+2' });
  });
});
```

---

## 20.6 测试配置与最佳实践

### 20.6.1 Vitest 配置示例

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    environment: 'node',
    globals: true,
    setupFiles: ['./__tests__/setup.ts'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
      exclude: ['node_modules', '__tests__', '**/*.d.ts'],
    },
    // Mock Anthropic SDK 默认行为
    mock: [
      ['@anthropic-ai/sdk', { default: {} }],
    ],
    // 测试超时
    testTimeout: 10000,
  },
});
```

```typescript
// __tests__/setup.ts
// 全局测试设置
import { vi } from 'vitest';

// Mock fs 模块（避免测试中意外的文件操作）
vi.mock('fs', () => ({
  existsSync: vi.fn(() => true),
  readFileSync: vi.fn(() => '{}'),
  writeFileSync: vi.fn(),
  mkdirSync: vi.fn(),
}));
```

### 20.6.2 环境变量与配置隔离

```typescript
// __tests__/setup.ts 的补充
process.env.ANTHROPIC_API_KEY = 'test-key-for-unit-tests';
process.env.NODE_ENV = 'test';
```

```typescript
// .env.test
ANTHROPIC_API_KEY=test-key
LOG_LEVEL=silent
MOCK_API_BASE_URL=http://localhost:3141
```

### 20.6.3 CI 中的测试策略

```yaml
# .github/workflows/test.yml（GitHub Actions 示例）
name: Tests

on: [push, pull_request]

jobs:
  unit-and-integration:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run test:unit -- --coverage
      - run: npm run test:integration

  # E2E 测试只在 main 分支和 PR 时运行
  e2e:
    if: github.ref == 'refs/heads/main' || github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: npm run test:e2e
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

### 20.6.4 测试覆盖率目标

| 测试类型 | 覆盖率目标 | 关键检查点 |
|---------|-----------|-----------|
| Prompt 模板 | >95% | 所有分支、所有参数组合 |
| Tool 解析 | >90% | 必填/可选参数、Schema 验证 |
| SDK 调用封装 | >80% | 正常路径、错误路径 |
| 业务逻辑 | >85% | 边界条件、空输入 |

---

## 20.7 本章小结

本章我们学了四种 Mock 策略，从简单到复杂：

| 策略 | 适用场景 | 复杂度 | 真实性 |
|------|---------|--------|--------|
| **Mock SDK 方法** | 单元测试、快速验证 | ⭐ | ⭐⭐ |
| **Mock HTTP Server** | 集成测试、错误场景 | ⭐⭐⭐ | ⭐⭐⭐⭐ |
| **MSW** | 浏览器/前端测试 | ⭐⭐ | ⭐⭐⭐⭐ |
| **VCR 录制回放** | 回归测试、真实响应 | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |

**老三的忠告**：测试写不写是态度问题，怎么写是能力问题。Mock 策略没有银弹，根据场景选对工具才是真本事。

---

> 💡 **下一章预告**：第21章我们已经覆盖了。第22章我们将深入 **缓存策略与性能调优**，看看 Prompt Caching、多层缓存、语义缓存怎么用。
