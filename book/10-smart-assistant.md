# 第10章：实战案例——智能助手

> "前面九章练的内功，今天全部打出来。" —— 老三

学完了消息、工具、MCP、多轮对话、配置文件……你可能觉得知识点都掌握了，但真要写一个完整的项目，还是不知道从哪下手。这一章，我们从零构建一个**智能开发助手**，把前面所有知识点串起来。

这个助手能做什么？

- 📁 **读取项目文件**——让 Claude 了解你的代码
- 🔍 **搜索代码**——快速定位函数、变量
- 🏃 **执行命令**——跑测试、装依赖
- 💬 **多轮对话**——记住上下文，持续交互
- 📝 **自动记录**——把重要决策写入记忆文件

---

## 10.1 需求分析

### 10.1.1 场景描述

想象你是一个开发者，打开终端想问 Claude：

```
> 帮我看看 src/index.ts 里有没有未处理的异常
> 给 run() 函数加上 try-catch
> 跑一下测试，看结果
> 把刚才的修改总结一下，写到 CHANGES.md
```

这就是我们的目标：一个能理解项目、操作文件、执行命令、保持上下文的智能助手。

### 10.1.2 功能拆解

| 功能 | 对应 SDK 能力 | 章节 |
|------|-------------|------|
| 读取文件 | Tool（read_file） | 第6章 |
| 搜索代码 | Tool（search_code） | 第6章 |
| 执行命令 | Tool（run_command） | 第6章 |
| 写入文件 | Tool（write_file） | 第6章 |
| 多轮上下文 | Conversation History | 第9章 |
| 项目配置 | CLAUDE.md | 第8章 |

### 10.1.3 架构设计

```
┌─────────────────────────────────────┐
│           用户交互层 (REPL)          │
│    输入 → 处理 → 展示 → 循环         │
├─────────────────────────────────────┤
│           SDK 调用层                 │
│    query() + tools + conversation    │
├─────────────────────────────────────┤
│           工具执行层                 │
│  read_file | search_code | ...      │
├─────────────────────────────────────┤
│           文件系统                   │
│  项目目录 / 沙箱目录                 │
└─────────────────────────────────────┘
```

---

## 10.2 项目搭建

### 10.2.1 初始化项目

```bash
mkdir dev-assistant && cd dev-assistant
npm init -y
npm install @anthropic-ai/claude-code
npm install -D typescript @types/node
npx tsc --init
```

### 10.2.2 项目结构

```
dev-assistant/
├── src/
│   ├── index.ts          # 入口 + REPL 循环
│   ├── assistant.ts      # 核心助手逻辑
│   ├── tools.ts          # 工具定义与执行
│   └── utils.ts          # 辅助函数
├── CLAUDE.md             # 项目级配置
├── tsconfig.json
└── package.json
```

### 10.2.3 CLAUDE.md 配置

在项目根目录创建 `CLAUDE.md`，让助手了解自己的角色：

```markdown
# Dev Assistant

你是一个智能开发助手，帮助用户理解代码、修改文件、执行命令。

## 规则
- 修改文件前先展示 diff
- 执行命令前要确认（危险操作）
- 回答要简洁，关键信息加粗
- 中文回复
```

---

## 10.3 工具定义与实现

### 10.3.1 工具定义（tools.ts）

```typescript
import { Tool } from '@anthropic-ai/claude-code';
import * as fs from 'fs';
import * as path from 'path';
import { execSync } from 'child_process';

// ========== 工具定义 ==========

export const readFileTool: Tool = {
  name: 'read_file',
  description: '读取指定路径的文件内容。返回文件的全部文本。',
  input_schema: {
    type: 'object' as const,
    properties: {
      path: {
        type: 'string',
        description: '文件路径（相对于项目根目录）',
      },
    },
    required: ['path'],
  },
};

export const searchCodeTool: Tool = {
  name: 'search_code',
  description: '在项目中搜索包含指定关键词的文件。返回匹配的文件路径和行号。',
  input_schema: {
    type: 'object' as const,
    properties: {
      query: {
        type: 'string',
        description: '搜索关键词或正则表达式',
      },
      extension: {
        type: 'string',
        description: '限定文件扩展名，如 "ts"、"py"。默认搜索所有文件。',
      },
    },
    required: ['query'],
  },
};

export const writeFileTool: Tool = {
  name: 'write_file',
  description: '将内容写入指定文件。如果文件存在则覆盖，不存在则创建。',
  input_schema: {
    type: 'object' as const,
    properties: {
      path: {
        type: 'string',
        description: '文件路径（相对于项目根目录）',
      },
      content: {
        type: 'string',
        description: '要写入的文件内容',
      },
    },
    required: ['path', 'content'],
  },
};

export const runCommandTool: Tool = {
  name: 'run_command',
  description: '在项目目录下执行 shell 命令，返回标准输出。注意：危险操作（如 rm、git push）需要用户确认。',
  input_schema: {
    type: 'object' as const,
    properties: {
      command: {
        type: 'string',
        description: '要执行的 shell 命令',
      },
    },
    required: ['command'],
  },
};

export const listFilesTool: Tool = {
  name: 'list_files',
  description: '列出指定目录下的文件和子目录。默认列出项目根目录。',
  input_schema: {
    type: 'object' as const,
    properties: {
      dir: {
        type: 'string',
        description: '目录路径（相对于项目根目录），默认为 "."',
      },
    },
  },
};

// ========== 工具执行 ==========

// 项目根目录——根据实际情况修改
const PROJECT_ROOT = process.cwd();

function resolvePath(relativePath: string): string {
  const resolved = path.resolve(PROJECT_ROOT, relativePath);
  // 安全检查：防止路径遍历攻击
  if (!resolved.startsWith(PROJECT_ROOT)) {
    throw new Error(`路径越界：${relativePath}`);
  }
  return resolved;
}

export function executeTool(name: string, input: Record<string, any>): string {
  switch (name) {
    case 'read_file': {
      const filePath = resolvePath(input.path);
      if (!fs.existsSync(filePath)) {
        return `错误：文件不存在 - ${input.path}`;
      }
      const content = fs.readFileSync(filePath, 'utf-8');
      // 限制返回长度，避免 token 爆炸
      const MAX_CHARS = 10_000;
      if (content.length > MAX_CHARS) {
        return content.slice(0, MAX_CHARS) + '\n\n... (文件过长，已截断)';
      }
      return content;
    }

    case 'search_code': {
      const query = input.query;
      const ext = input.extension || '';
      try {
        // 用 grep 实现，简单高效
        let cmd = `grep -rn "${query}" ${PROJECT_ROOT}`;
        if (ext) {
          cmd += ` --include="*.${ext}"`;
        }
        cmd += ' --exclude-dir=node_modules --exclude-dir=.git';
        const result = execSync(cmd, {
          encoding: 'utf-8',
          maxBuffer: 1024 * 1024,
          timeout: 5000,
        });
        const lines = result.split('\n').slice(0, 30); // 最多30条结果
        return lines.join('\n') || '没有找到匹配结果';
      } catch (e: any) {
        // grep 没匹配时返回 exit code 1
        if (e.status === 1) return '没有找到匹配结果';
        return `搜索出错：${e.message}`;
      }
    }

    case 'write_file': {
      const filePath = resolvePath(input.path);
      const dir = path.dirname(filePath);
      if (!fs.existsSync(dir)) {
        fs.mkdirSync(dir, { recursive: true });
      }
      fs.writeFileSync(filePath, input.content, 'utf-8');
      return `✅ 文件已写入：${input.path}（${input.content.length} 字符）`;
    }

    case 'run_command': {
      const command = input.command;
      // 危险命令检查
      const dangerousPatterns = ['rm -rf', 'git push', 'DROP TABLE', 'mkfs', 'dd if='];
      const isDangerous = dangerousPatterns.some(p => command.includes(p));
      if (isDangerous) {
        return `⚠️ 检测到危险命令：${command}\n请手动确认后再执行。`;
      }
      try {
        const result = execSync(command, {
          encoding: 'utf-8',
          cwd: PROJECT_ROOT,
          maxBuffer: 1024 * 1024,
          timeout: 30_000,
        });
        return result || '（命令执行成功，无输出）';
      } catch (e: any) {
        return `命令执行失败（exit code ${e.status}）：\n${e.stderr || e.message}`;
      }
    }

    case 'list_files': {
      const dir = resolvePath(input.dir || '.');
      if (!fs.existsSync(dir)) {
        return `错误：目录不存在 - ${input.dir || '.'}`;
      }
      const entries = fs.readdirSync(dir, { withFileTypes: true });
      return entries
        .map(e => `${e.isDirectory() ? '📁' : '📄'} ${e.name}`)
        .join('\n');
    }

    default:
      return `未知工具：${name}`;
  }
}

// 所有工具的集合，传给 SDK 用
export const allTools: Tool[] = [
  readFileTool,
  searchCodeTool,
  writeFileTool,
  runCommandTool,
  listFilesTool,
];
```

**关键设计决策：**

1. **路径安全**：`resolvePath()` 防止 `../../etc/passwd` 这种路径遍历
2. **长度限制**：`read_file` 截断超长文件，避免 token 溢出
3. **危险命令拦截**：`run_command` 对 `rm -rf` 等命令做提醒
4. **搜索限制**：`search_code` 最多返回 30 条，排除 node_modules

---

## 10.4 核心助手逻辑

### 10.4.1 助手类（assistant.ts）

```typescript
import { query, type Message } from '@anthropic-ai/claude-code';
import { allTools, executeTool } from './tools.js';

export class DevAssistant {
  private conversationHistory: Message[] = [];

  /**
   * 处理用户输入，返回助手回复
   */
  async chat(userInput: string): Promise<string> {
    // 1. 把用户消息加入历史
    this.conversationHistory.push({
      role: 'user',
      content: userInput,
    });

    // 2. 调用 Claude Code SDK
    const response = await query({
      prompt: userInput,
      options: {
        model: 'claude-sonnet-4-20250514',
        maxTurns: 10, // 最多10轮工具调用
      },
    });

    // 3. 处理工具调用循环
    // SDK 会自动处理 tool_use → tool_result 的循环
    // 我们只需要拿到最终结果

    const assistantMessage = response.messages
      .filter((m: Message) => m.role === 'assistant')
      .pop();

    if (assistantMessage) {
      this.conversationHistory.push(assistantMessage);
    }

    // 提取文本内容
    return this.extractText(assistantMessage);
  }

  /**
   * 从 Message 中提取文本内容
   */
  private extractText(message?: Message): string {
    if (!message) return '（无回复）';

    if (typeof message.content === 'string') {
      return message.content;
    }

    // content 是 ContentBlock 数组
    return (message.content as any[])
      .filter(block => block.type === 'text')
      .map(block => block.text)
      .join('\n');
  }

  /**
   * 获取对话历史（用于调试）
   */
  getHistory(): Message[] {
    return [...this.conversationHistory];
  }

  /**
   * 清空对话历史
   */
  clearHistory(): void {
    this.conversationHistory = [];
  }
}
```

### 10.4.2 重要说明：SDK 的工具调用模式

Claude Code SDK 的 `query()` 函数内部已经**自动处理了工具调用循环**——它会：

1. 发送 prompt + tools 给 Claude
2. 收到 tool_use → 执行工具 → 返回 tool_result
3. 重复直到 Claude 不再调用工具，给出最终文本回复
4. 返回所有消息

所以你不需要手动写 `while (hasToolUse)` 循环！SDK 帮你搞定了。

但如果你想**自定义工具执行逻辑**（比如加日志、加确认），可以用底层 API：

```typescript
import { query } from '@anthropic-ai/claude-code';

// 使用 onToolCall 回调来自定义工具执行
const response = await query({
  prompt: '读取 package.json',
  options: {
    maxTurns: 10,
    tools: allTools,
    // 自定义工具执行器
    toolExecutor: async (toolName: string, input: Record<string, any>) => {
      console.log(`[工具调用] ${toolName}`, input);
      const result = executeTool(toolName, input);
      console.log(`[工具结果]`, result.slice(0, 100));
      return result;
    },
  },
});
```

---

## 10.5 REPL 交互界面

### 10.5.1 入口文件（index.ts）

```typescript
import * as readline from 'readline';
import { DevAssistant } from './assistant.js';

async function main() {
  const assistant = new DevAssistant();
  const rl = readline.createInterface({
    input: process.stdin,
    output: process.stdout,
  });

  console.log('🤖 Dev Assistant 已启动！');
  console.log('输入你的问题，按回车发送。输入 /quit 退出，/clear 清空上下文。\n');

  const askQuestion = (): void => {
    rl.question('你 > ', async (input: string) => {
      const trimmed = input.trim();

      // 命令处理
      if (trimmed === '/quit' || trimmed === '/exit') {
        console.log('👋 再见！');
        rl.close();
        return;
      }
      if (trimmed === '/clear') {
        assistant.clearHistory();
        console.log('🧹 对话历史已清空\n');
        askQuestion();
        return;
      }
      if (!trimmed) {
        askQuestion();
        return;
      }

      // 调用助手
      try {
        const reply = await assistant.chat(trimmed);
        console.log(`\n助手 > ${reply}\n`);
      } catch (error: any) {
        console.error(`\n❌ 出错了：${error.message}\n`);
      }

      askQuestion();
    });
  };

  askQuestion();
}

main().catch(console.error);
```

### 10.5.2 编译运行

```bash
npx tsc
node dist/index.js
```

---

## 10.6 运行效果演示

### 10.6.1 探索项目

```
你 > 这个项目有什么文件？

助手 > 让我看看项目结构。

[工具调用] list_files {"dir": "."}
[工具结果] 📁 src
📄 package.json
📄 tsconfig.json
📄 CLAUDE.md
📄 README.md

项目包含以下内容：
- **src/** — 源代码目录
- **package.json** — 项目配置
- **tsconfig.json** — TypeScript 配置
- **CLAUDE.md** — Claude 配置
- **README.md** — 项目说明
```

### 10.6.2 阅读代码

```
你 > 帮我看看 src/index.ts 里有没有未处理的异常

助手 > 让我读一下这个文件。

[工具调用] read_file {"path": "src/index.ts"}
[工具结果] import * as readline from 'readline';
...

检查完毕，发现以下问题：

1. **第23行**：`JSON.parse(input)` 没有 try-catch，如果输入不是合法 JSON 会抛异常
2. **第31行**：`fs.readFileSync(configPath)` 没有处理文件不存在的情况

建议加 try-catch 包裹，需要我帮你改吗？
```

### 10.6.3 执行命令

```
你 > 跑一下测试

助手 > 好的，执行测试。

[工具调用] run_command {"command": "npm test"}
[工具结果] 
> dev-assistant@1.0.0 test
> jest

 PASS  src/utils.test.ts
  ✓ formatDate returns ISO string (3ms)
  ✓ parseConfig handles valid JSON (1ms)
  ✕ parseConfig handles invalid JSON (5ms)

Tests: 1 failed, 2 passed, 3 total

有 **1个测试失败**：`parseConfig handles invalid JSON`。这正好对应我刚才提到的 JSON.parse 没有异常处理的问题。修好那个 bug，测试就能过了。
```

### 10.6.4 持续对话

```
你 > 帮我修一下那个 bug

助手 > 好的，我来修复 src/index.ts 中的 JSON.parse 异常处理。

[工具调用] write_file {"path": "src/index.ts", "content": "..."}
[工具结果] ✅ 文件已写入：src/index.ts（1234 字符）

已修复！改动：
- 给 `JSON.parse` 加了 try-catch
- 给 `readFileSync` 加了文件存在性检查

再跑一次测试验证？

你 > 跑一下

助手 > 
[工具调用] run_command {"command": "npm test"}
[工具结果] Tests: 3 passed, 3 total

✅ **全部通过！** Bug 修好了。
```

---

## 10.7 进阶：添加 MCP Server

我们的助手已经很强了，但还可以通过 MCP 接入更多能力。比如加一个 GitHub MCP Server，让助手能操作 PR 和 Issue。

### 10.7.1 配置 MCP

在项目目录创建 `.mcp.json`：

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "你的Token"
      }
    }
  }
}
```

### 10.7.2 在 SDK 中启用 MCP

```typescript
import { query } from '@anthropic-ai/claude-code';

const response = await query({
  prompt: '查看这个仓库的最近5个 PR',
  options: {
    maxTurns: 10,
    tools: allTools,
    // MCP Server 会自动从 .mcp.json 加载
    // 也可以手动指定：
    mcpServers: {
      github: {
        command: 'npx',
        args: ['-y', '@modelcontextprotocol/server-github'],
        env: {
          GITHUB_PERSONAL_ACCESS_TOKEN: process.env.GITHUB_TOKEN!,
        },
      },
    },
  },
});
```

这样助手就能直接操作 GitHub 了——查看 PR、创建 Issue、搜索代码……全部通过自然语言驱动。

---

## 10.8 完整代码清单

整个项目的核心代码加起来不到 300 行。以下是文件清单：

| 文件 | 行数 | 说明 |
|------|------|------|
| `src/tools.ts` | ~160 | 工具定义与执行 |
| `src/assistant.ts` | ~70 | 助手核心逻辑 |
| `src/index.ts` | ~50 | REPL 入口 |
| `CLAUDE.md` | ~10 | 项目配置 |
| **合计** | **~290** | |

完整代码可在项目目录直接运行：

```bash
# 克隆并运行
git clone https://github.com/example/dev-assistant.git
cd dev-assistant
npm install
npx tsc
node dist/index.js
```

---

## 10.9 小结

这一章我们干了不少事：

1. **需求分析** → 明确了智能助手的核心功能
2. **架构设计** → 四层架构，职责清晰
3. **工具实现** → 5个核心工具，覆盖文件、搜索、命令
4. **安全设计** → 路径遍历防护、危险命令拦截、长度限制
5. **REPL 界面** → 交互式命令行，支持 /quit /clear
6. **MCP 扩展** → 通过 MCP Server 接入 GitHub 能力

核心收获：**Claude Code SDK 的 query() 自动处理工具调用循环**，你只需要定义工具和执行逻辑，SDK 帮你搞定往返通信。这让构建智能助手的代码量从"几千行"降到"几百行"。

下一章，我们用类似的思路构建一个**代码生成工具**——这次重点在 Prompt 工程和流式输出。

---

> "从 'Hello World' 到智能助手，你只差一个 query()。" —— 老三
