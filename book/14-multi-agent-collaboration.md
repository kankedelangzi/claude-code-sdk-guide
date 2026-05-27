# 第14章：多 Agent 协作

> **本章目标**：学会用 Claude Code SDK 构建多 Agent 协作系统，让多个 AI Agent 像团队一样分工合作、完成复杂任务。

---

## 14.1 为什么需要多 Agent 协作？

### 14.1.1 单 Agent 的局限

想象一下，如果让一个 Claude 实例去完成"开发一个完整的 Web 应用"这个任务：

- 它需要同时处理需求分析、架构设计、代码编写、测试、部署...
- 上下文窗口会被海量信息填满
- 没有专业分工，每个环节都做得不够深入
- 无法并行处理多个子任务

**单 Agent = 全能但低效的"独行侠"**

### 14.1.2 多 Agent 的优势

多 Agent 协作就像组建了一个"AI 团队"：

```
┌─────────────────────────────────────────────────┐
│              Orchestrator Agent                  │
│           (任务拆解、协调、汇总)                 │
└────────────┬────────────────────┬───────────────┘
             │                    │
    ┌────────▼────────┐  ┌───────▼──────────┐
    │  Coder Agent    │  │  Tester Agent    │
    │  (写代码)       │  │  (写测试)        │
    └─────────────────┘  └──────────────────┘
```

**优势：**
- **专业分工**：每个 Agent 专注一个领域
- **并行执行**：多个 Agent 同时工作
- **上下文隔离**：每个 Agent 只关注自己的任务
- **可扩展性**：需要新能力？加个新 Agent 就行

---

## 14.2 Claude Code SDK 中的多 Agent 架构

### 14.2.1 核心概念

在 Claude Code SDK 中，多 Agent 协作涉及以下核心概念：

| 概念 | 说明 | 对应 SDK API |
|------|------|--------------|
| **Orchestrator** | 主 Agent，负责任务拆解和协调 | `messages.create()` + Tool 定义 |
| **Sub-Agent** | 子 Agent，执行具体任务 | 独立的 `messages.create()` 调用 |
| **Task** | 分配给 Agent 的工作单元 | 自定义的 task 数据结构 |
| **Message Passing** | Agent 间通信方式 | 通过 `content` 字段传递 |
| **Shared State** | Agent 间共享的状态 | 外部存储（文件/数据库） |

### 14.2.2 Claude Code 的 Sub-Agent 机制

根据 Claude Code 的源码分析，Sub-Agent 的完整生命周期包括：

1. **身份生成** (`createAgentId()`)：为每个 Sub-Agent 生成唯一 ID
2. **工具池裁剪**：根据 Sub-Agent 的角色分配不同的工具
3. **System Prompt 组装**：为每个 Sub-Agent 定制系统提示词
4. **AbortController 隔离**：每个 Sub-Agent 有独立的终止控制
5. **进入 `query()` 循环**：Sub-Agent 开始工作

---

## 14.3 实战：构建一个多 Agent 协作系统

### 14.3.1 场景定义

我们要构建一个"智能代码审查系统"，包含以下 Agent：

- **Orchestrator Agent**：接收代码，拆解审查任务
- **Security Agent**：检查安全漏洞
- **Performance Agent**：分析性能问题
- **Style Agent**：检查代码风格
- **Report Agent**：汇总审查报告

### 14.3.2 实现代码

```typescript
// multi-agent-code-review.ts
import Anthropic from '@anthropic-ai/sdk';
import { MessageParam } from '@anthropic-ai/sdk/resources/messages';

const client = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY,
});

// ============ 类型定义 ============

interface Task {
  id: string;
  type: 'security' | 'performance' | 'style' | 'report';
  code: string;
  status: 'pending' | 'running' | 'completed' | 'failed';
  result?: string;
  dependencies?: string[]; // 依赖的其他 task ID
}

interface AgentConfig {
  name: string;
  systemPrompt: string;
  tools?: any[];
}

// ============ Agent 定义 ============

const AGENTS: Record<string, AgentConfig> = {
  orchestrator: {
    name: 'Orchestrator',
    systemPrompt: `你是任务编排 Agent。你的工作是：
1. 接收用户提交的代码
2. 将代码审查任务拆解为子任务（安全、性能、风格）
3. 协调子 Agent 执行审查
4. 汇总结果并生成最终报告

重要：你必须返回 JSON 格式的任务列表。`,
  },
  security: {
    name: 'Security Agent',
    systemPrompt: `你是安全审查专家。请仔细检查代码中的：
- SQL 注入、XSS 等安全漏洞
- 不安全的 API 调用
- 敏感信息泄露
- 权限校验缺失

只关注安全问题，不要评论其他方面。`,
  },
  performance: {
    name: 'Performance Agent',
    systemPrompt: `你是性能优化专家。请分析代码中的：
- 时间复杂度问题
- 空间复杂度问题
- 不必要的计算和内存占用
- 并发和异步处理问题

只关注性能问题，不要评论其他方面。`,
  },
  style: {
    name: 'Style Agent',
    systemPrompt: `你是代码风格审查专家。请检查：
- 命名规范（变量、函数、类名）
- 代码格式（缩进、空格、换行）
- 注释完整性
- 代码结构和可读性

只关注代码风格，不要评论其他方面。`,
  },
  report: {
    name: 'Report Agent',
    systemPrompt: `你是报告生成专家。请将各 Agent 的审查结果汇总成一份完整的代码审查报告。
报告应包括：
1. 总体评价
2. 安全问题（如有）
3. 性能问题（如有）
4. 代码风格问题（如有）
5. 改进建议`,
  },
};

// ============ Agent 执行器 ============

class AgentExecutor {
  private client: Anthropic;
  
  constructor(client: Anthropic) {
    this.client = client;
  }
  
  async runAgent(
    agentKey: string,
    task: Task,
    context?: string // 来自其他 Agent 的结果
  ): Promise<string> {
    const agent = AGENTS[agentKey];
    if (!agent) throw new Error(`Unknown agent: ${agentKey}`);
    
    console.log(`🤖 [${agent.name}] 开始执行任务 ${task.id}`);
    task.status = 'running';
    
    let userMessage = `请审查以下代码：\n\n\`\`\`\n${task.code}\n\`\`\``;
    
    // 如果有上下文（其他 Agent 的结果），附加进去
    if (context) {
      userMessage += `\n\n参考信息：\n${context}`;
    }
    
    try {
      const response = await this.client.messages.create({
        model: 'claude-sonnet-4-20250514',
        max_tokens: 4096,
        system: agent.systemPrompt,
        messages: [
          { role: 'user', content: userMessage }
        ],
      });
      
      const result = response.content
        .filter(block => block.type === 'text')
        .map(block => (block as any).text)
        .join('\n');
      
      task.status = 'completed';
      task.result = result;
      
      console.log(`✅ [${agent.name}] 任务 ${task.id} 完成`);
      return result;
      
    } catch (error) {
      task.status = 'failed';
      console.error(`❌ [${agent.name}] 任务 ${task.id} 失败:`, error);
      throw error;
    }
  }
}

// ============ 任务编排器 ============

class Orchestrator {
  private executor: AgentExecutor;
  private tasks: Map<string, Task> = new Map();
  
  constructor(executor: AgentExecutor) {
    this.executor = executor;
  }
  
  // 步骤1：拆解任务
  async decomposeTask(code: string): Promise<Task[]> {
    console.log('📋 Orchestrator: 拆解任务...\n');
    
    const tasks: Task[] = [
      {
        id: 'security-review',
        type: 'security',
        code,
        status: 'pending',
      },
      {
        id: 'performance-review',
        type: 'performance',
        code,
        status: 'pending',
      },
      {
        id: 'style-review',
        type: 'style',
        code,
        status: 'pending',
      },
      {
        id: 'generate-report',
        type: 'report',
        code,
        status: 'pending',
        dependencies: ['security-review', 'performance-review', 'style-review'],
      },
    ];
    
    tasks.forEach(task => this.tasks.set(task.id, task));
    return tasks;
  }
  
  // 步骤2：执行任务（支持依赖管理）
  async executeTask(task: Task): Promise<void> {
    // 检查依赖是否完成
    if (task.dependencies) {
      for (const depId of task.dependencies) {
        const dep = this.tasks.get(depId);
        if (!dep || dep.status !== 'completed') {
          throw new Error(`依赖任务 ${depId} 未完成`);
        }
      }
    }
    
    // 执行任务
    const agentKey = task.type; // security -> security agent
    await this.executor.runAgent(agentKey, task);
  }
  
  // 步骤3：并行执行无依赖的任务
  async executeParallel(tasks: Task[]): Promise<void> {
    // 先执行无依赖的任务
    const independentTasks = tasks.filter(t => !t.dependencies);
    console.log(`🚀 并行执行 ${independentTasks.length} 个独立任务...\n`);
    
    await Promise.all(
      independentTasks.map(task => this.executeTask(task))
    );
    
    // 执行有依赖的任务
    const dependentTasks = tasks.filter(t => t.dependencies);
    for (const task of dependentTasks) {
      await this.executeTask(task);
    }
  }
  
  // 获取任务结果
  getTaskResult(taskId: string): string | undefined {
    return this.tasks.get(taskId)?.result;
  }
}

// ============ 主流程 ============

async function codeReview(code: string) {
  console.log('🔍 开始代码审查...\n');
  
  const executor = new AgentExecutor(client);
  const orchestrator = new Orchestrator(executor);
  
  // 步骤1：拆解任务
  const tasks = await orchestrator.decomposeTask(code);
  
  // 步骤2：执行任务
  await orchestrator.executeParallel(tasks);
  
  // 步骤3：生成最终报告
  const securityResult = orchestrator.getTaskResult('security-review');
  const performanceResult = orchestrator.getTaskResult('performance-review');
  const styleResult = orchestrator.getTaskResult('style-review');
  
  console.log('\n📊 所有审查完成，生成最终报告...\n');
  
  // 把子任务结果传给 Report Agent
  const finalReport = await executor.runAgent('report', {
    id: 'final-report',
    type: 'report',
    code,
    status: 'running',
    result: undefined,
    dependencies: [],
  }, `
## 安全审查结果：
${securityResult}

## 性能审查结果：
${performanceResult}

## 代码风格审查结果：
${styleResult}
  `);
  
  console.log('\n✅ 代码审查完成！\n');
  console.log('=' .repeat(60));
  console.log(finalReport);
  console.log('=' .repeat(60));
  
  return finalReport;
}

// ============ 测试代码 ============

const sampleCode = `
import express from 'express';
import { db } from './database';

const app = express();

app.post('/login', (req, res) => {
  const { username, password } = req.body;
  
  // 查询用户（有 SQL 注入风险！）
  const user = db.query(\`SELECT * FROM users WHERE username='\${username}' AND password='\${password}'\`);
  
  if (user) {
    res.json({ token: 'abc123', userId: user.id });
  } else {
    res.status(401).json({ error: 'Login failed' });
  }
});

app.get('/users', (req, res) => {
  // 没有权限检查！
  const users = db.query('SELECT * FROM users');
  res.json(users);
});

app.listen(3000);
`;

// 运行代码审查
codeReview(sampleCode).catch(console.error);
```

### 14.3.3 运行结果示例

```
🔍 开始代码审查...

📋 Orchestrator: 拆解任务...

🚀 并行执行 3 个独立任务...

🤖 [Security Agent] 开始执行任务 security-review
🤖 [Performance Agent] 开始执行任务 performance-review
🤖 [Style Agent] 开始执行任务 style-review
✅ [Security Agent] 任务 security-review 完成
✅ [Performance Agent] 任务 performance-review 完成
✅ [Style Agent] 任务 style-review 完成

📊 所有审查完成，生成最终报告...

🤖 [Report Agent] 开始执行任务 final-report
✅ [Report Agent] 任务 final-report 完成

✅ 代码审查完成！

============================================================
# 代码审查报告

## 总体评价
代码存在严重安全问题，需要立即修复。性能方面没有明显问题，但代码风格需要改进。

## 安全问题（严重）
1. **SQL 注入漏洞**：直接拼接用户输入到 SQL 语句中
2. **敏感信息泄露**：密码明文存储和比较
3. **缺少身份验证**：/users 端点没有权限检查

## 性能问题
无明显性能问题。

## 代码风格问题
1. 缺少输入验证
2. 没有错误处理
3. 代码格式需要统一

## 改进建议
1. 使用参数化查询防止 SQL 注入
2. 使用 bcrypt 等安全方式存储密码
3. 添加 JWT 或其他身份验证机制
4. 添加输入验证和错误处理
============================================================
```

---

## 14.4 高级模式：Agent 间通信

### 14.4.1 通过 Tools 实现 Agent 间调用

Claude Code SDK 的 Tool Use 机制可以让 Agent 调用其他 Agent：

```typescript
// agent-tools.ts

// 定义一个"调用其他 Agent"的 Tool
const callAgentTool = {
  name: 'call_agent',
  description: '调用另一个 Agent 来执行子任务',
  input_schema: {
    type: 'object',
    properties: {
      agent_type: {
        type: 'string',
        enum: ['security', 'performance', 'style', 'report'],
        description: '要调用的 Agent 类型',
      },
      task_description: {
        type: 'string',
        description: '任务描述',
      },
    },
    required: ['agent_type', 'task_description'],
  },
};

// Orchestrator Agent 使用这个 Tool
async function orchestratorWithTools(code: string) {
  const response = await client.messages.create({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 4096,
    system: AGENTS.orchestrator.systemPrompt,
    messages: [
      { 
        role: 'user', 
        content: `请审查以下代码：\n\`\`\`\n${code}\n\`\`\`` 
      }
    ],
    tools: [callAgentTool],
  });
  
  // 处理 Tool Use
  for (const block of response.content) {
    if (block.type === 'tool_use' && block.name === 'call_agent') {
      const { agent_type, task_description } = block.input;
      
      console.log(`🔧 调用 ${agent_type} Agent: ${task_description}`);
      
      // 执行对应的 Agent
      const result = await executor.runAgent(agent_type, {
        id: `tool-call-${Date.now()}`,
        type: agent_type,
        code,
        status: 'pending',
      });
      
      // 把结果返回给 Orchestrator
      // ... (省略 Tool Result 的处理)
    }
  }
}
```

### 14.4.2 通过共享状态通信

```typescript
// shared-state.ts

class SharedState {
  private state: Map<string, any> = new Map();
  
  // Agent A 写入状态
  set(key: string, value: any) {
    this.state.set(key, value);
  }
  
  // Agent B 读取状态
  get(key: string): any {
    return this.state.get(key);
  }
  
  // 等待某个状态出现（用于 Agent 间同步）
  async waitFor(key: string, timeoutMs: number = 30000): Promise<any> {
    const start = Date.now();
    
    while (Date.now() - start < timeoutMs) {
      if (this.state.has(key)) {
        return this.state.get(key);
      }
      await new Promise(resolve => setTimeout(resolve, 100));
    }
    
    throw new Error(`等待超时: ${key}`);
  }
}

// 使用共享状态
const sharedState = new SharedState();

// Agent A
async function agentA() {
  const result = await doSomeWork();
  sharedState.set('agentA_result', result);
}

// Agent B (依赖 Agent A 的结果)
async function agentB() {
  const agentAResult = await sharedState.waitFor('agentA_result');
  const finalResult = await process(agentAResult);
  sharedState.set('final_result', finalResult);
}
```

---

## 14.5 最佳实践

### 14.5.1 Agent 设计原则

1. **单一职责**：每个 Agent 只做一件事
2. **明确输入输出**：定义清晰的接口
3. **无状态设计**：Agent 不保存状态，状态放在外部
4. **可观测性**：为每个 Agent 添加日志

### 14.5.2 错误处理

```typescript
class RobustAgentExecutor extends AgentExecutor {
  async runAgentWithRetry(
    agentKey: string,
    task: Task,
    maxRetries: number = 3
  ): Promise<string> {
    let lastError: Error | null = null;
    
    for (let i = 0; i < maxRetries; i++) {
      try {
        return await this.runAgent(agentKey, task);
      } catch (error) {
        lastError = error as Error;
        console.warn(`⚠️  第 ${i + 1} 次重试...`);
        await new Promise(resolve => setTimeout(resolve, 1000 * Math.pow(2, i)));
      }
    }
    
    throw lastError;
  }
}
```

### 14.5.3 成本控制

```typescript
class CostAwareOrchestrator extends Orchestrator {
  private totalTokens: number = 0;
  private maxTokens: number = 100000; // 预算上限
  
  async executeTask(task: Task): Promise<void> {
    if (this.totalTokens > this.maxTokens) {
      throw new Error('超出 Token 预算！');
    }
    
    // ... 执行任务
    
    // 统计 Token 使用
    this.totalTokens += response.usage.input_tokens + response.usage.output_tokens;
    console.log(`💰 已使用 ${this.totalTokens} tokens`);
  }
}
```

---

## 14.6 实战案例：自动化软件开发流水线

这个案例展示如何用多 Agent 协作完成一个完整的软件开发任务：

```typescript
// sdlc-pipeline.ts (Software Development Life Cycle)

enum PipelineStage {
  REQUIREMENTS = 'requirements',
  DESIGN = 'design',
  CODING = 'coding',
  TESTING = 'testing',
  REVIEW = 'review',
  DEPLOYMENT = 'deployment',
}

class SDLCPipeline {
  private stages: Map<PipelineStage, any> = new Map();
  
  constructor() {
    // 初始化各个阶段的 Agent
    this.stages.set(PipelineStage.REQUIREMENTS, new RequirementsAgent());
    this.stages.set(PipelineStage.DESIGN, new DesignAgent());
    this.stages.set(PipelineStage.CODING, new CodingAgent());
    this.stages.set(PipelineStage.TESTING, new TestingAgent());
    this.stages.set(PipelineStage.REVIEW, new ReviewAgent());
    this.stages.set(PipelineStage.DEPLOYMENT, new DeploymentAgent());
  }
  
  async run(userRequirement: string): Promise<void> {
    console.log('🚀 启动 SDLC 流水线...\n');
    
    let currentOutput = userRequirement;
    
    // 顺序执行各个阶段
    for (const stage of Object.values(PipelineStage)) {
      console.log(`\n📍 阶段: ${stage}`);
      console.log('-' .repeat(40));
      
      const agent = this.stages.get(stage);
      currentOutput = await agent.execute(currentOutput);
    }
    
    console.log('\n✅ SDLC 流水线执行完成！');
  }
}

// 运行
const pipeline = new SDLCPipeline();
pipeline.run('开发一个待办事项管理系统').catch(console.error);
```

---

## 14.7 小结

本章我们学习了：

1. **为什么需要多 Agent 协作**：突破单 Agent 的局限
2. **Claude Code SDK 的多 Agent 架构**：Orchestrator + Sub-Agent
3. **如何实现多 Agent 协作**：
   - 任务拆解与编排
   - 并行执行
   - Agent 间通信
4. **最佳实践**：单一职责、错误处理、成本控制
5. **实战案例**：代码审查系统、SDLC 流水线

**关键要点：**
- 用 Orchestrator 拆解任务
- 用 Sub-Agent 执行专项任务
- 通过 Tools 或共享状态实现 Agent 间通信
- 始终考虑错误处理和成本控制

---

## 练习题

1. 修改代码审查系统，添加一个 `Documentation Agent`，专门检查代码注释和 README 完整性。

2. 实现一个"翻译流水线"：将英文文档翻译成中文、日文、法文三个版本，用三个并行 Agent 执行。

3. 为多 Agent 系统添加"人工审批"环节：当 Security Agent 发现严重漏洞时，暂停流水线，等待人工确认后再继续。

---

**下一章**：第15章《生产环境部署》—— 当你准备好把 Agent 系统投入生产时，这一章会教你如何做。

---

*写于 2026-05-20，Claude Code SDK 版本：1.0.0*
