# 第11章：实战案例——代码生成工具

> "让 Claude 不只是回答问题，而是直接帮你写代码、存文件、跑测试。" —— 老三

第10章我们构建了一个智能助手，能读文件、搜代码、执行命令。这一章，我们要更进一步——构建一个**代码生成工具**，它能根据自然语言描述自动生成代码、写入文件、执行测试，形成完整的"需求→代码→验证"闭环。

这个工具能做什么？

- 📝 **需求解析** —— 把自然语言需求拆解成技术任务
- 🏗️ **代码生成** —— 根据 Schema 和模板生成结构化代码
- 💾 **文件写入** —— 自动把生成的代码保存到正确位置
- 🧪 **测试验证** —— 运行测试确认生成代码的正确性
- 🔄 **迭代修正** —— 测试失败时自动修复并重新验证

---

## 11.1 需求分析

### 11.1.1 场景描述

你是一个前端开发者，经常需要写重复性的代码：

```
> 帮我生成一个 React 用户登录表单组件，包含邮箱和密码字段，带表单验证
> 给我写一个 Express REST API 的 CRUD 路由，模型是 Product，字段有 name、price、stock
> 生成一个 TypeScript 工具函数库，包含 debounce、throttle、deepClone
```

手动写这些代码虽然不难，但非常耗时。我们的目标是：**一句话搞定，代码直接落盘，测试跑通才算完**。

### 11.1.2 核心功能拆解

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   需求解析    │───▶│   代码生成    │───▶│   文件写入    │───▶│   测试验证    │
│  (理解意图)   │    │  (Claude 写码) │    │  (保存到磁盘)  │    │  (跑测试)     │
└──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
                                                                   │
                     ┌──────────────┐                              │
                     │   迭代修正    │◀─────────────────────────────┘
                     │  (失败则修复)  │
                     └──────────────┘
```

---

## 11.2 技术实现

### 11.2.1 项目结构

```
code-generator/
├── src/
│   ├── index.ts          # 入口文件
│   ├── tools.ts          # 工具定义
│   ├── generator.ts      # 代码生成核心逻辑
│   └── prompts.ts        # 系统提示词
├── output/               # 生成的代码输出目录
├── package.json
└── tsconfig.json
```

### 11.2.2 工具定义

代码生成工具需要三个核心 Tool：

```typescript
// src/tools.ts

/** Tool 1：生成代码 —— 核心 Tool，让 Claude 根据需求写代码 */
export const generateCodeTool = {
  name: 'generate_code',
  description: '根据需求描述生成代码。返回包含文件路径和代码内容的结构化结果。',
  input_schema: {
    type: 'object' as const,
    properties: {
      requirement: {
        type: 'string',
        description: '需求描述，例如：生成一个 React 登录表单组件',
      },
      language: {
        type: 'string',
        description: '目标编程语言，如 typescript、python、go',
        enum: ['typescript', 'javascript', 'python', 'go', 'rust'],
      },
      framework: {
        type: 'string',
        description: '使用的框架，如 react、express、fastapi',
      },
      output_path: {
        type: 'string',
        description: '输出文件路径，相对于 output 目录',
      },
    },
    required: ['requirement', 'language'],
  },
};

/** Tool 2：写入文件 —— 将生成的代码保存到磁盘 */
export const writeFileTool = {
  name: 'write_file',
  description: '将代码内容写入指定文件路径。如果文件已存在则覆盖。',
  input_schema: {
    type: 'object' as const,
    properties: {
      path: {
        type: 'string',
        description: '文件路径（相对于项目根目录）',
      },
      content: {
        type: 'string',
        description: '文件内容',
      },
    },
    required: ['path', 'content'],
  },
};

/** Tool 3：运行命令 —— 执行测试、格式化等命令 */
export const runCommandTool = {
  name: 'run_command',
  description: '在终端执行 shell 命令，用于运行测试、格式化代码等。',
  input_schema: {
    type: 'object' as const,
    properties: {
      command: {
        type: 'string',
        description: '要执行的 shell 命令',
      },
      timeout: {
        type: 'number',
        description: '超时时间（毫秒），默认 30000',
      },
    },
    required: ['command'],
  },
};
```

### 11.2.3 系统提示词

提示词是代码生成质量的关键。我们需要让 Claude 明白：**你是一个代码生成专家，不是聊天机器人**。

```typescript
// src/prompts.ts

export const CODE_GENERATOR_SYSTEM_PROMPT = `你是一个专业的代码生成专家。你的任务是根据用户需求生成高质量、可直接运行的代码。

## 工作流程

1. 分析用户需求，理解要生成什么
2. 使用 generate_code 工具生成代码
3. 使用 write_file 工具将代码写入文件
4. 使用 run_command 工具运行测试或格式化（如果适用）
5. 如果测试失败，分析错误并修复代码

## 代码质量要求

- 代码必须可直接运行，不能有语法错误
- 包含必要的类型定义（TypeScript）或类型注释（Python）
- 包含错误处理
- 遵循最佳实践和常见代码风格
- 必要时添加注释，但不啰嗦

## 输出规范

- 每次生成代码后，必须用 write_file 写入文件
- 文件路径遵循项目约定
- 如果用户没有指定框架，选择最合适的
- 生成代码的同时，尽量生成对应的测试文件`;
```

### 11.2.4 核心生成逻辑

这是整个项目的核心——处理 Claude 的 Tool Call，执行实际操作，并把结果返回给 Claude 继续推理。

```typescript
// src/generator.ts

import Anthropic from '@anthropic-ai/sdk';
import * as fs from 'fs';
import * as path from 'path';
import { execSync } from 'child_process';
import { generateCodeTool, writeFileTool, runCommandTool } from './tools';
import { CODE_GENERATOR_SYSTEM_PROMPT } from './prompts';

const client = new Anthropic();

/** 工具执行器：根据 tool_use 调用执行实际操作 */
function executeTool(name: string, input: Record<string, unknown>): string {
  switch (name) {
    case 'generate_code': {
      // generate_code 是一个"信号"Tool，告诉 Claude 该生成代码了
      // 实际代码由 Claude 在后续回复中通过 write_file 输出
      const { requirement, language, framework } = input;
      return JSON.stringify({
        status: 'ready',
        message: `准备生成代码：${requirement}`,
        language,
        framework: framework || 'auto-detect',
        hint: '请在下一条消息中使用 write_file 工具写入生成的代码',
      });
    }

    case 'write_file': {
      const filePath = input.path as string;
      const content = input.content as string;

      // 确保目录存在
      const dir = path.dirname(filePath);
      if (!fs.existsSync(dir)) {
        fs.mkdirSync(dir, { recursive: true });
      }

      fs.writeFileSync(filePath, content, 'utf-8');
      return JSON.stringify({
        status: 'success',
        message: `文件已写入：${filePath}`,
        size: content.length,
      });
    }

    case 'run_command': {
      const command = input.command as string;
      const timeout = (input.timeout as number) || 30000;

      try {
        const output = execSync(command, {
          timeout,
          encoding: 'utf-8',
          cwd: process.cwd(),
        });
        return JSON.stringify({
          status: 'success',
          output: output.slice(0, 5000), // 截断过长输出
        });
      } catch (error: unknown) {
        const err = error as { stdout?: string; stderr?: string; message?: string };
        return JSON.stringify({
          status: 'error',
          output: (err.stdout || '').slice(0, 3000),
          error: (err.stderr || err.message || '').slice(0, 3000),
        });
      }
    }

    default:
      return JSON.stringify({ status: 'error', message: `未知工具：${name}` });
  }
}

/** 主循环：持续与 Claude 对话直到任务完成 */
export async function generateCode(userRequirement: string): Promise<void> {
  const tools = [generateCodeTool, writeFileTool, runCommandTool];

  const messages: Anthropic.MessageParam[] = [
    {
      role: 'user',
      content: userRequirement,
    },
  ];

  let continueLoop = true;
  let iteration = 0;
  const MAX_ITERATIONS = 10; // 防止无限循环

  while (continueLoop && iteration < MAX_ITERATIONS) {
    iteration++;
    console.log(`\n🔄 迭代 #${iteration}`);

    const response = await client.messages.create({
      model: 'claude-sonnet-4-20250514',
      maxTokens: 4096,
      system: CODE_GENERATOR_SYSTEM_PROMPT,
      tools,
      messages,
    });

    // 收集 assistant 回复
    const assistantContent: Anthropic.ContentBlockParam[] = [];
    const toolResults: Anthropic.ToolResultBlockParam[] = [];

    for (const block of response.content) {
      if (block.type === 'text') {
        console.log(`💬 Claude: ${block.text.slice(0, 200)}...`);
        assistantContent.push({ type: 'text', text: block.text });
      } else if (block.type === 'tool_use') {
        console.log(`🔧 调用工具: ${block.name}`);
        console.log(`   参数: ${JSON.stringify(block.input).slice(0, 200)}...`);

        // 执行工具
        const result = executeTool(block.name, block.input as Record<string, unknown>);
        console.log(`   结果: ${result.slice(0, 200)}...`);

        assistantContent.push({
          type: 'tool_use',
          id: block.id,
          name: block.name,
          input: block.input,
        });

        toolResults.push({
          type: 'tool_result',
          tool_use_id: block.id,
          content: result,
        });
      }
    }

    // 如果没有 tool_use，说明 Claude 认为任务完成了
    if (toolResults.length === 0) {
      continueLoop = false;
      console.log('\n✅ 任务完成！');
      break;
    }

    // 把 assistant 回复和 tool_result 加入消息历史
    messages.push({ role: 'assistant', content: assistantContent });
    messages.push({ role: 'user', content: toolResults });
  }

  if (iteration >= MAX_ITERATIONS) {
    console.log('\n⚠️ 达到最大迭代次数，停止生成');
  }
}
```

### 11.2.5 入口文件

```typescript
// src/index.ts

import { generateCode } from './generator';

async function main() {
  const requirement = process.argv.slice(2).join(' ');

  if (!requirement) {
    console.log('用法: npx ts-node src/index.ts "你的需求描述"');
    console.log('');
    console.log('示例:');
    console.log('  npx ts-node src/index.ts "生成一个 Express REST API 的 CRUD 路由，模型是 Product"');
    console.log('  npx ts-node src/index.ts "生成一个 React 登录表单组件，带表单验证"');
    process.exit(1);
  }

  console.log(`🚀 开始生成代码：${requirement}`);
  await generateCode(requirement);
}

main().catch(console.error);
```

---

## 11.3 效果展示

### 11.3.1 示例1：生成 Express CRUD 路由

```bash
$ npx ts-node src/index.ts \
  "生成一个 Express REST API 的 CRUD 路由，模型是 Product，字段有 name(string)、price(number)、stock(number)，包含输入验证和错误处理"

🚀 开始生成代码：生成一个 Express REST API 的 CRUD 路由...

🔄 迭代 #1
💬 Claude: 我来为你生成一个完整的 Express CRUD 路由...
🔧 调用工具: write_file
   参数: {"path":"output/routes/products.ts","content":"import express..."}
   结果: {"status":"success","message":"文件已写入：output/routes/products.ts"}

🔄 迭代 #2
🔧 调用工具: write_file
   参数: {"path":"output/routes/products.test.ts","content":"import request..."}
   结果: {"status":"success","message":"文件已写入：output/routes/products.test.ts"}

🔄 迭代 #3
🔧 调用工具: run_command
   参数: {"command":"npx ts-node output/routes/products.test.ts"}
   结果: {"status":"success","output":"✓ 5 tests passed"}

✅ 任务完成！
```

### 11.3.2 生成结果：products.ts

以下是 Claude 生成的代码（经过格式化）：

```typescript
// output/routes/products.ts

import express, { Request, Response, NextFunction } from 'express';

interface Product {
  id: string;
  name: string;
  price: number;
  stock: number;
}

// 内存存储（实际项目请替换为数据库）
const products: Map<string, Product> = new Map();
let idCounter = 1;

const router = express.Router();

// 输入验证中间件
function validateProduct(req: Request, res: Response, next: NextFunction): void {
  const { name, price, stock } = req.body;
  const errors: string[] = [];

  if (!name || typeof name !== 'string' || name.trim().length === 0) {
    errors.push('name 必须是非空字符串');
  }
  if (typeof price !== 'number' || price < 0) {
    errors.push('price 必须是非负数');
  }
  if (typeof stock !== 'number' || !Number.isInteger(stock) || stock < 0) {
    errors.push('stock 必须是非负整数');
  }

  if (errors.length > 0) {
    res.status(400).json({ errors });
    return;
  }
  next();
}

// GET /products —— 获取所有产品
router.get('/', (_req: Request, res: Response) => {
  res.json(Array.from(products.values()));
});

// GET /products/:id —— 获取单个产品
router.get('/:id', (req: Request, res: Response) => {
  const product = products.get(req.params.id);
  if (!product) {
    res.status(404).json({ error: '产品不存在' });
    return;
  }
  res.json(product);
});

// POST /products —— 创建产品
router.post('/', validateProduct, (req: Request, res: Response) => {
  const id = String(idCounter++);
  const product: Product = { id, ...req.body };
  products.set(id, product);
  res.status(201).json(product);
});

// PUT /products/:id —— 更新产品
router.put('/:id', validateProduct, (req: Request, res: Response) => {
  if (!products.has(req.params.id)) {
    res.status(404).json({ error: '产品不存在' });
    return;
  }
  const product: Product = { id: req.params.id, ...req.body };
  products.set(req.params.id, product);
  res.json(product);
});

// DELETE /products/:id —— 删除产品
router.delete('/:id', (req: Request, res: Response) => {
  if (!products.has(req.params.id)) {
    res.status(404).json({ error: '产品不存在' });
    return;
  }
  products.delete(req.params.id);
  res.status(204).end();
});

export default router;
```

### 11.3.3 示例2：生成 React 表单组件

```bash
$ npx ts-node src/index.ts \
  "生成一个 React 登录表单组件，使用 TypeScript，包含邮箱和密码字段，邮箱格式验证，密码最少8位，提交后显示 loading 状态"

🚀 开始生成代码：生成一个 React 登录表单组件...

🔄 迭代 #1
🔧 调用工具: write_file
   参数: {"path":"output/components/LoginForm.tsx",...}
   结果: {"status":"success","message":"文件已写入：output/components/LoginForm.tsx"}

✅ 任务完成！
```

生成结果（核心部分）：

```typescript
// output/components/LoginForm.tsx

import React, { useState, FormEvent } from 'react';

interface LoginFormProps {
  onSubmit: (email: string, password: string) => Promise<void>;
}

interface FormErrors {
  email?: string;
  password?: string;
}

export function LoginForm({ onSubmit }: LoginFormProps) {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [errors, setErrors] = useState<FormErrors>({});
  const [loading, setLoading] = useState(false);
  const [success, setSuccess] = useState(false);

  function validate(): FormErrors {
    const errs: FormErrors = {};
    if (!email) {
      errs.email = '邮箱不能为空';
    } else if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) {
      errs.email = '邮箱格式不正确';
    }
    if (!password) {
      errs.password = '密码不能为空';
    } else if (password.length < 8) {
      errs.password = '密码至少需要8位';
    }
    return errs;
  }

  async function handleSubmit(e: FormEvent) {
    e.preventDefault();
    const validationErrors = validate();
    setErrors(validationErrors);

    if (Object.keys(validationErrors).length > 0) return;

    setLoading(true);
    try {
      await onSubmit(email, password);
      setSuccess(true);
    } catch {
      setErrors({ email: '登录失败，请重试' });
    } finally {
      setLoading(false);
    }
  }

  if (success) {
    return <div className="success">✅ 登录成功！</div>;
  }

  return (
    <form onSubmit={handleSubmit}>
      <div>
        <label htmlFor="email">邮箱</label>
        <input
          id="email"
          type="email"
          value={email}
          onChange={(e) => setEmail(e.target.value)}
          disabled={loading}
        />
        {errors.email && <span className="error">{errors.email}</span>}
      </div>
      <div>
        <label htmlFor="password">密码</label>
        <input
          id="password"
          type="password"
          value={password}
          onChange={(e) => setPassword(e.target.value)}
          disabled={loading}
        />
        {errors.password && <span className="error">{errors.password}</span>}
      </div>
      <button type="submit" disabled={loading}>
        {loading ? '登录中...' : '登录'}
      </button>
    </form>
  );
}
```

---

## 11.4 进阶：支持模板和约束

上面的例子是"自由生成"——Claude 根据需求自由发挥。但在实际项目中，我们往往需要约束生成的代码符合特定的模板或规范。

### 11.4.1 带模板的代码生成

```typescript
// src/generator-advanced.ts

/** 代码模板，用于约束生成格式 */
interface CodeTemplate {
  name: string;
  description: string;
  // 模板中的占位符，Claude 会填充这些变量
  variables: string[];
  // 参考代码（可选），让 Claude 模仿风格
  example?: string;
}

/** React 组件模板 */
const reactComponentTemplate: CodeTemplate = {
  name: 'react-component',
  description: 'React 函数组件，使用 TypeScript，包含 Props 类型定义',
  variables: ['componentName', 'props', 'state', 'effects', 'render'],
  example: `
import React from 'react';

interface {{componentName}}Props {
  // 在此定义 Props
}

export function {{componentName}}(props: {{componentName}}Props) {
  return <div>{{componentName}}</div>;
}
`.trim(),
};

/** Express 路由模板 */
const expressRouteTemplate: CodeTemplate = {
  name: 'express-crud-route',
  description: 'Express REST API 的 CRUD 路由，包含输入验证和错误处理',
  variables: ['modelName', 'fields', 'validations'],
};

/** 根据模板构建增强版提示词 */
function buildTemplatePrompt(template: CodeTemplate, requirement: string): string {
  let prompt = CODE_GENERATOR_SYSTEM_PROMPT;
  prompt += `\n\n## 代码模板约束\n`;
  prompt += `当前使用模板：${template.name} - ${template.description}\n`;
  prompt += `必须包含以下部分：${template.variables.join('、')}\n`;

  if (template.example) {
    prompt += `\n### 参考示例\n\`\`\`\n${template.example}\n\`\`\`\n`;
    prompt += `请参照上面的示例风格生成代码，但要适配用户的具体需求。\n`;
  }

  return prompt;
}

/** 使用模板生成代码 */
export async function generateWithTemplate(
  requirement: string,
  template: CodeTemplate,
): Promise<void> {
  const systemPrompt = buildTemplatePrompt(template, requirement);
  const tools = [generateCodeTool, writeFileTool, runCommandTool];

  const messages: Anthropic.MessageParam[] = [
    { role: 'user', content: requirement },
  ];

  let continueLoop = true;
  let iteration = 0;

  while (continueLoop && iteration < 10) {
    iteration++;

    const response = await client.messages.create({
      model: 'claude-sonnet-4-20250514',
      maxTokens: 4096,
      system: systemPrompt,
      tools,
      messages,
    });

    const assistantContent: Anthropic.ContentBlockParam[] = [];
    const toolResults: Anthropic.ToolResultBlockParam[] = [];

    for (const block of response.content) {
      if (block.type === 'text') {
        assistantContent.push({ type: 'text', text: block.text });
      } else if (block.type === 'tool_use') {
        const result = executeTool(block.name, block.input as Record<string, unknown>);

        assistantContent.push({
          type: 'tool_use',
          id: block.id,
          name: block.name,
          input: block.input,
        });

        toolResults.push({
          type: 'tool_result',
          tool_use_id: block.id,
          content: result,
        });
      }
    }

    if (toolResults.length === 0) {
      continueLoop = false;
      break;
    }

    messages.push({ role: 'assistant', content: assistantContent });
    messages.push({ role: 'user', content: toolResults });
  }
}
```

### 11.4.2 运行模板生成

```bash
# 使用 React 组件模板
$ npx ts-node src/index.ts \
  --template react-component \
  "生成一个 UserProfile 组件，展示用户头像、姓名、简介，支持编辑模式切换"
```

模板的核心价值：**让生成结果更可预测、更一致**。没有模板，Claude 可能每次用不同的代码风格；有模板，输出始终符合团队规范。

---

## 11.5 关键设计决策

### 11.5.1 为什么用 Tool 而不是直接让 Claude 输出代码？

| 方案 | 优点 | 缺点 |
|------|------|------|
| 直接输出文本 | 简单 | 无法执行后续操作（写文件、跑测试） |
| Tool Call 循环 | 可执行实际操作、可迭代修正 | 实现稍复杂 |

我们选择 Tool Call 循环，因为**代码生成不只是写代码，还包括验证和修正**。

### 11.5.2 为什么 generate_code 是"信号"Tool？

你可能注意到 `generate_code` 工具的执行器只返回了一个"ready"状态，并没有真正生成代码。这是因为：

1. **代码生成是 Claude 的能力**，不是外部工具的功能
2. `generate_code` 是一个"触发器"，告诉 Claude "是时候生成代码了"
3. Claude 会在后续回复中通过 `write_file` 把代码写入文件

这种模式叫**信号 Tool（Signal Tool）**——不执行实际操作，而是引导 Claude 的行为流程。

### 11.5.3 迭代修正：自动修复失败测试

```typescript
// 在系统提示词中增加修复指导
const FIX_PROMPT = `
如果测试失败，请按以下步骤修复：
1. 仔细阅读错误信息
2. 定位错误原因（语法错误、类型错误、逻辑错误）
3. 修改代码并重新写入文件
4. 重新运行测试
5. 最多修复 3 次，超过 3 次则报告失败原因
`;
```

---

## 11.6 本章小结

本章我们构建了一个完整的代码生成工具，核心要点：

1. **三个核心 Tool**：`generate_code`（信号触发）、`write_file`（写文件）、`run_command`（执行命令）
2. **Tool Call 循环**：Claude 调用工具 → 执行 → 返回结果 → Claude 继续推理，直到任务完成
3. **信号 Tool 模式**：有些 Tool 不执行操作，而是引导 Claude 的行为
4. **模板约束**：通过模板让生成结果更可预测、更一致
5. **迭代修正**：测试失败时自动修复，形成闭环

下一章，我们将讨论性能优化与最佳实践，让这些工具在真实项目中跑得更快、更稳。

---

_老三注：代码生成工具的核心不是"生成"本身，而是"生成→验证→修正"的闭环。没有验证的代码生成，跟 Stack Overflow 复制粘贴没区别。_
