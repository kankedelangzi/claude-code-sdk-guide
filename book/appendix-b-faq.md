# 附录B：常见问题解答

> "遇到问题不可怕，可怕的是不知道怎么解决。" —— 老三

这个附录收集了 Claude Code SDK 开发过程中最常见的问题，按类别组织。每个问题都附有原因分析和解决方案，还有可运行的代码示例。

---

## B.1 安装问题

### B.1.1 `command not found: claude`

**现象：** 安装完成后运行 `claude` 命令，提示找不到命令。

```bash
$ claude
command not found: claude
```

**原因：** 安装目录不在 PATH 环境变量中。

**解决方案：**

```bash
# macOS/Linux - 检查安装目录是否在 PATH
echo $PATH | tr ':' '\n' | grep -Fx "$HOME/.local/bin"

# 如果没有输出，添加到 shell 配置
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc  # Zsh
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc # Bash

# 重新加载配置
source ~/.zshrc  # 或 source ~/.bashrc

# 验证
claude --version
```

**Windows PowerShell：**

```powershell
# 检查 PATH
$env:PATH -split ';' | Select-String '\.local\\bin'

# 如果没有输出，添加到用户 PATH
$currentPath = [Environment]::GetEnvironmentVariable('PATH', 'User')
[Environment]::SetEnvironmentVariable('PATH', "$currentPath;$env:USERPROFILE\.local\bin", 'User')

# 重启终端后验证
claude --version
```

---

### B.1.2 权限错误 `Permission denied`

**现象：** 安装时提示权限不足。

```bash
curl: Failed to create file '/usr/local/bin/claude': Permission denied
```

**原因：** 目标目录需要管理员权限。

**解决方案：**

```bash
# 方案1：使用 sudo（不推荐长期使用）
sudo curl -fsSL https://claude.ai/install.sh | bash

# 方案2：安装到用户目录（推荐）
mkdir -p ~/.local/bin
export INSTALL_DIR="$HOME/.local/bin"
curl -fsSL https://claude.ai/install.sh | bash
```

---

### B.1.3 安装脚本返回 HTML 而非脚本

**现象：** 运行安装命令后报语法错误。

```bash
curl -fsSL https://claude.ai/install.sh | bash
# 报错：syntax error near unexpected token '<'
```

**原因：** 网络问题（如公司代理、防火墙）导致返回了 HTML 错误页面而非安装脚本。

**解决方案：**

```bash
# 1. 检查网络连接
curl -sI https://downloads.claude.ai/claude-code-releases/latest

# 2. 如果需要代理
export HTTP_PROXY=http://proxy.example.com:8080
export HTTPS_PROXY=http://proxy.example.com:8080
curl -fsSL https://claude.ai/install.sh | bash

# 3. 或者使用 Homebrew（macOS）
brew install --cask claude-code
```

---

### B.1.4 Windows 上的 shell 问题

**现象：** 在 Windows 上运行安装命令报错。

```powershell
# 错误1：'irm' is not recognized
# 原因：你在 CMD 中，应该用 PowerShell

# 错误2：The token '&&' is not a valid statement separator
# 原因：你在 PowerShell 中使用了 CMD 语法
```

**解决方案：**

```powershell
# PowerShell（推荐）
irm https://claude.ai/install.ps1 | iex

# CMD
curl -fsSL https://claude.ai/install.cmd -o install.cmd && install.cmd && del install.cmd

# WinGet
winget install Anthropic.ClaudeCode
```

**提示：** PowerShell 提示符显示 `PS C:\>`，CMD 只显示 `C:\>`。

---

### B.1.5 TLS/SSL 证书错误

**现象：** 安装时提示证书验证失败。

```bash
curl: (60) SSL certificate problem: unable to get local issuer certificate
```

**原因：** 系统 CA 证书过期或公司代理使用自签名证书。

**解决方案：**

```bash
# macOS - 更新证书
brew install ca-certificates
brew link --force ca-certificates

# Linux (Debian/Ubuntu)
sudo apt update && sudo apt install --reinstall ca-certificates

# 临时跳过验证（仅调试用，别在生产环境这么干！）
export NODE_TLS_REJECT_UNAUTHORIZED=0
# ⚠️ 上面的操作会让你裸奔，调试完赶紧改回来
```

---

## B.2 认证与配置问题

### B.2.1 `ANTHROPIC_API_KEY` 未设置

**现象：** 运行 SDK 报 401 认证失败。

```
Error: Authentication failed. Please set ANTHROPIC_API_KEY environment variable.
```

**原因：** 没有设置 API Key 环境变量，或者 Key 已过期/无效。

**解决方案：**

```bash
# 1. 设置环境变量（临时）
export ANTHROPIC_API_KEY="sk-ant-api03-xxxxxxxxxxxxx"

# 2. 写入 shell 配置（永久）
echo 'export ANTHROPIC_API_KEY="sk-ant-api03-xxxxxxxxxxxxx"' >> ~/.zshrc
source ~/.zshrc

# 3. 或者在代码中直接传入
```

```javascript
import { ClaudeCode } from '@anthropic-ai/claude-code';

const sdk = new ClaudeCode({
  apiKey: process.env.ANTHROPIC_API_KEY, // 从环境变量读取
});
```

**提示：** 别把 Key 硬编码在代码里然后推到 GitHub —— 别问我怎么知道的 🙃

---

### B.2.2 API Key 权限不足

**现象：** 认证通过但调用某些功能报 403。

```
Error: You do not have permission to access this resource.
```

**原因：** API Key 对应的账户没有对应功能的访问权限（如 Model Access、Tool Use 等）。

**解决方案：**

1. 登录 [Anthropic Console](https://console.anthropic.com/) 检查 API Key 权限
2. 确认账户已开通对应模型的访问权限
3. 如使用组织 Key，确认组织管理员已授权

---

### B.2.3 配置文件不生效

**现象：** 修改了 `CLAUDE.md` 或 `.claude/settings.json`，但行为没有变化。

**原因：**
- 配置文件位置不对
- JSON 格式有误
- 优先级被其他配置覆盖

**解决方案：**

```bash
# 检查配置文件加载顺序（优先级从高到低）
# 1. 命令行参数
# 2. 环境变量
# 3. 项目级 .claude/settings.json
# 4. 用户级 ~/.claude/settings.json
# 5. CLAUDE.md / CLAUDE.local.md

# 验证 JSON 格式
cat .claude/settings.json | python3 -m json.tool

# 查看实际生效的配置
claude config list
```

---

## B.3 运行时错误

### B.3.1 Tool call limit exceeded

**现象：** 单轮对话中 Tool 调用次数超限，SDK 抛出错误。

```
Error: Tool call limit exceeded: maximum 128 tool calls per turn
```

**原因：** SDK 默认每轮对话最多允许 128 次 Tool 调用。你的 Agent 陷入了循环调用或者确实需要更多调用。

**解决方案：**

```javascript
import { ClaudeCode } from '@anthropic-ai/claude-code';

// 方案1：调大限制（慎重，先想想是不是逻辑有问题）
const sdk = new ClaudeCode({
  maxTurns: 50,            // 最大对话轮数
  maxToolCalls: 256,       // 单轮最大 Tool 调用次数
});

// 方案2：优化 Agent 逻辑，避免循环调用
// 常见坑：Agent 读文件 → 发现不够 → 再读 → 还是不够 → ……无限循环
// 解决：在 prompt 里明确说「最多读 3 次文件，读不到就停下来报告」

// 方案3：用 abort controller 做兜底
const controller = new AbortController();
const timeout = setTimeout(() => controller.abort(), 60_000); // 60秒超时

try {
  const result = await sdk.query('帮我重构这个项目', {
    signal: controller.signal,
  });
} finally {
  clearTimeout(timeout);
}
```

**老三经验：** 90% 的 limit exceeded 都是因为 Agent 在循环。先检查你的 prompt，再考虑调大限制。

---

### B.3.2 Context window overflow

**现象：** 对话上下文超出模型窗口限制。

```
Error: Request too large: input tokens (210000) exceed model context window (200000)
```

**原因：** 对话历史 + Tool 返回结果太大，撑爆了上下文窗口。

**解决方案：**

```javascript
import { ClaudeCode } from '@anthropic-ai/claude-code';

// 方案1：限制 Tool 返回的内容长度
const sdk = new ClaudeCode({
  toolOptions: {
    readFile: {
      maxOutputLength: 5000,  // 限制文件读取返回的最大字符数
    },
    search: {
      maxResults: 20,         // 限制搜索结果数量
    },
  },
});

// 方案2：手动管理对话历史，截断旧消息
function trimMessages(messages, maxTokens = 180000) {
  // 保留系统提示和最近的消息，中间的摘要
  // 简单粗暴版：只保留最近 N 条
  const systemMessages = messages.filter(m => m.role === 'system');
  const otherMessages = messages.filter(m => m.role !== 'system');
  const trimmed = otherMessages.slice(-20); // 保留最近 20 轮
  return [...systemMessages, ...trimmed];
}

// 方案3：使用 SDK 内置的上下文压缩
const result = await sdk.query('继续之前的重构', {
  contextManagement: {
    compressionEnabled: true,   // 开启上下文压缩
    maxContextTokens: 180000,   // 保留一些余量
  },
});
```

**老三提示：** 就像收拾行李箱 —— 与其买更大的箱子，不如学会断舍离。

---

### B.3.3 Rate limit reached

**现象：** API 请求频率超限。

```
Error: Rate limit reached for anthropic/claude-sonnet-4-20250514
Retry after 20 seconds
```

**原因：** 短时间内发送了太多请求，触发了 Anthropic 的速率限制。

**解决方案：**

```javascript
import { ClaudeCode } from '@anthropic-ai/claude-code';

// 方案1：内置重试（推荐）
const sdk = new ClaudeCode({
  maxRetries: 3,              // 自动重试次数
  retryDelayMs: 1000,         // 初始重试延迟（指数退避）
});

// 方案2：手动实现指数退避
async function queryWithRetry(prompt, maxRetries = 5) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await sdk.query(prompt);
    } catch (error) {
      if (error.status !== 429) throw error; // 不是限流错误，直接抛
      
      const retryAfter = error.headers?.['retry-after']
        ? parseInt(error.headers['retry-after']) * 1000
        : Math.min(1000 * Math.pow(2, i), 60000); // 指数退避，最多等 60 秒
      
      console.log(`限流了，${retryAfter / 1000} 秒后重试 (${i + 1}/${maxRetries})`);
      await new Promise(resolve => setTimeout(resolve, retryAfter));
    }
  }
  throw new Error('重试次数耗尽，还是限流 😢');
}

// 方案3：并发控制（多任务场景）
import { PQueue } from 'p-queue'; // npm install p-queue

const queue = new PQueue({
  concurrency: 2,          // 同时最多 2 个请求
  interval: 1000,          // 每秒最多 2 个
  intervalCap: 2,
});

const results = await Promise.all(
  prompts.map(p => queue.add(() => sdk.query(p)))
);
```

**老三心得：** 限流不是 Bug，是 Anthropic 在保护你也保护大家。优雅地退避，别硬刚。

---

### B.3.4 Tool 执行超时

**现象：** Tool 调用长时间无响应，最终超时。

```
Error: Tool execution timed out after 30000ms
```

**原因：** Tool 执行耗时超过默认超时时间，常见于：
- 执行耗时 Shell 命令
- 读取大文件
- 网络请求卡住

**解决方案：**

```javascript
import { ClaudeCode } from '@anthropic-ai/claude-code';

// 方案1：调整超时时间
const sdk = new ClaudeCode({
  toolOptions: {
    bash: {
      timeout: 120_000,     // Shell 命令超时 120 秒
    },
    readFile: {
      timeout: 60_000,      // 文件读取超时 60 秒
    },
  },
});

// 方案2：在 Tool 实现中自己做超时控制
import { exec } from 'child_process';
import { promisify } from 'util';

const execAsync = promisify(exec);

async function safeExec(command, timeoutMs = 30_000) {
  const controller = new AbortController();
  const timer = setTimeout(() => controller.abort(), timeoutMs);
  
  try {
    const { stdout, stderr } = await execAsync(command, {
      signal: controller.signal,
      timeout: timeoutMs,
    });
    return { success: true, stdout, stderr };
  } catch (error) {
    if (error.killed) {
      return { 
        success: false, 
        error: `命令超时（${timeoutMs / 1000}秒），可能卡住了`,
      };
    }
    throw error;
  } finally {
    clearTimeout(timer);
  }
}

// 方案3：给耗时任务开后门 —— 用异步模式
const result = await sdk.query('运行测试套件', {
  toolOptions: {
    bash: {
      mode: 'async',          // 异步执行，不阻塞对话
      pollInterval: 5000,     // 每 5 秒检查一次状态
    },
  },
});
```

**老三提醒：** 超时 ≠ 失败。有时候命令还在跑，只是 SDK 不等了。异步模式是你的好朋友。

---

## B.4 SDK 使用问题

### B.4.1 import 报错 `Cannot find module '@anthropic-ai/claude-code'`

**现象：** TypeScript 或 Node.js 项目中 import SDK 失败。

```typescript
// TypeScript
import { ClaudeCode } from '@anthropic-ai/claude-code';
// 报错：Cannot find module '@anthropic-ai/claude-code'
// 或者：Could not find a declaration file for module
```

**原因：**
- 没安装 SDK 依赖
- TypeScript 版本不兼容
- 模块解析配置有误

**解决方案：**

```bash
# 1. 确认已安装
npm install @anthropic-ai/claude-code

# 2. 检查 package.json
cat package.json | grep claude-code
# 应该能看到："@anthropic-ai/claude-code": "^1.0.0"
```

```javascript
// 3. 如果用 CommonJS（老旧项目，懂的都懂）
const { ClaudeCode } = require('@anthropic-ai/claude-code');

// 4. 如果用 ESM（现代项目，老三推荐）
import { ClaudeCode } from '@anthropic-ai/claude-code';
// 确保 package.json 中有："type": "module"
```

```json
// 5. tsconfig.json 配置检查
{
  "compilerOptions": {
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "target": "ES2022",
    "esModuleInterop": true
  }
}
```

**老三吐槽：** 90% 的 import 问题都是没装包。剩下 10% 是 `type: "module"` 忘了加。

---

### B.4.2 Streaming 响应解析

**现象：** 使用 streaming 模式时，收到的数据不知道怎么解析。

```javascript
// 收到一堆 SSE 事件，不知道怎么处理
// event: content_block_start
// event: content_block_delta  
// event: content_block_stop
// event: message_stop
```

**原因：** Streaming 返回的是分块的 SSE 事件流，需要正确地拼接和处理。

**解决方案：**

```javascript
import { ClaudeCode } from '@anthropic-ai/claude-code';

// 方案1：使用 SDK 内置的流式处理
const sdk = new ClaudeCode();

const stream = sdk.query('解释量子计算', {
  stream: true,  // 开启流式响应
});

let fullResponse = '';

for await (const event of stream) {
  switch (event.type) {
    case 'content_block_start':
      // 新内容块开始（可能是文本、Tool Call 等）
      console.log('开始新内容块:', event.content_block.type);
      break;
      
    case 'content_block_delta':
      // 增量文本 —— 这就是打字机效果的核心！
      if (event.delta.type === 'text_delta') {
        process.stdout.write(event.delta.text);  // 实时输出
        fullResponse += event.delta.text;
      }
      break;
      
    case 'content_block_stop':
      // 内容块结束
      console.log('\n内容块结束');
      break;
      
    case 'message_stop':
      // 整个消息结束
      console.log('\n=== 完成 ===');
      break;
  }
}

console.log('完整回复:', fullResponse);

// 方案2：只想拿最终结果，不想处理流式细节
const result = await sdk.query('解释量子计算');
// result 就是完整结果，SDK 内部帮你处理了流式拼接
console.log(result.content);
```

**老三比喻：** Streaming 就像吃自助餐 —— 菜一道一道上，你可以边吃边等，也可以等齐了再开动。

---

### B.4.3 Tool result 格式错误

**现象：** 自定义 Tool 返回结果后，模型报错或行为异常。

```
Error: Tool result must be a string or array of content blocks
```

**原因：** Tool 的返回值格式不符合 SDK 要求。常见错误：
- 返回了 `undefined`
- 返回了复杂对象但没序列化
- 返回值类型不对

**解决方案：**

```javascript
import { ClaudeCode } from '@anthropic-ai/claude-code';

const sdk = new ClaudeCode({
  tools: [{
    name: 'calculate',
    description: '执行数学计算',
    input_schema: {
      type: 'object',
      properties: {
        expression: { type: 'string', description: '数学表达式' },
      },
      required: ['expression'],
    },
    // ✅ 正确：返回字符串
    execute: async ({ expression }) => {
      try {
        const result = eval(expression); // 别在生产环境用 eval！这里只是示例
        return `计算结果: ${expression} = ${result}`;
      } catch (error) {
        // ✅ 错误也要返回字符串，不要 throw
        return `计算失败: ${error.message}`;
      }
    },
  }],
});

// ❌ 错误示范
const badTool = {
  name: 'bad_calculate',
  execute: async ({ expression }) => {
    const result = eval(expression);
    return { result, expression }; // 对象不行！要字符串
    // return undefined;            // undefined 也不行！
    // return 42;                    // 数字也不行！
  },
};

// ✅ 如果一定要返回结构化数据，用 JSON 字符串
const structuredTool = {
  name: 'structured_calculate',
  execute: async ({ expression }) => {
    const result = eval(expression);
    return JSON.stringify({
      expression,
      result,
      timestamp: new Date().toISOString(),
    }, null, 2);  // 格式化输出，模型更容易理解
  },
};
```

**老三总结：** Tool 返回值就一条规矩 —— **必须是字符串**。记住了吗？

---

### B.4.4 Image/Vision 支持

**现象：** 想让模型分析图片，但不知道怎么传入图片。

```
// 图片怎么传给 Claude？直接给文件路径？URL？还是 base64？
```

**原因：** 图片需要以特定格式编码后作为消息内容传入。

**解决方案：**

```javascript
import { ClaudeCode } from '@anthropic-ai/claude-code';
import { readFileSync } from 'fs';

const sdk = new ClaudeCode();

// 方式1：base64 编码（本地图片）
const imageBuffer = readFileSync('./screenshot.png');
const base64Image = imageBuffer.toString('base64');

const result = await sdk.query([
  {
    role: 'user',
    content: [
      {
        type: 'image',
        source: {
          type: 'base64',
          media_type: 'image/png',  // 支持：image/png, image/jpeg, image/gif, image/webp
          data: base64Image,
        },
      },
      {
        type: 'text',
        text: '描述这张图片中的 UI 布局问题',
      },
    ],
  },
]);

// 方式2：URL（公开可访问的图片）
const result2 = await sdk.query([
  {
    role: 'user',
    content: [
      {
        type: 'image',
        source: {
          type: 'url',
          url: 'https://example.com/screenshot.png',
        },
      },
      {
        type: 'text',
        text: '这张图片有什么问题？',
      },
    ],
  },
]);
```

**支持的图片格式：**

| 格式 | media_type | 大小限制 |
|------|-----------|----------|
| PNG | `image/png` | ≤ 5MB |
| JPEG | `image/jpeg` | ≤ 5MB |
| GIF | `image/gif` | ≤ 5MB |
| WebP | `image/webp` | ≤ 5MB |

**老三提醒：** 图片超过 5MB？先压缩再传。别拿 4K 原图往里塞，模型不会因为你图片大就看得更清楚。

---

## B.5 MCP 协议问题

### B.5.1 MCP Server 连接失败

**现象：** 配置了 MCP Server，但 SDK 无法连接。

```
Error: Failed to connect to MCP server "my-tools": Connection refused
```

**原因：**
- MCP Server 没启动
- 端口配置错误
- 通信协议不匹配（stdio vs SSE）
- 权限问题

**解决方案：**

```bash
# 1. 手动测试 MCP Server 是否能启动
npx @my-org/mcp-server --help

# 2. 测试 stdio 模式的 MCP Server
echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test","version":"1.0"}}}' | npx @my-org/mcp-server
# 应该返回 initialize 响应
```

```javascript
// 3. 配置检查（stdio 模式 —— 最常见）
const sdk = new ClaudeCode({
  mcpServers: {
    'my-tools': {
      command: 'npx',
      args: ['-y', '@my-org/mcp-server'],
      // ⚠️ 如果 Server 在特定目录下才能运行
      cwd: '/path/to/server/dir',
      // ⚠️ 如果需要环境变量
      env: {
        API_KEY: process.env.MY_API_KEY,
      },
    },
  },
});

// 4. SSE 模式（远程 Server）
const sdk2 = new ClaudeCode({
  mcpServers: {
    'remote-tools': {
      url: 'http://localhost:3001/sse',  // SSE 端点
      // 如果需要认证头
      headers: {
        Authorization: `Bearer ${process.env.MCP_TOKEN}`,
      },
    },
  },
});
```

```bash
# 5. 查看当前已连接的 MCP Server
claude mcp list

# 6. 测试某个 MCP Server 的 Tool
claude mcp call my-tools some_tool '{"arg1": "value"}'
```

**老三排查顺序：** 先看 Server 能不能启动 → 再看 SDK 配置对不对 → 最后看网络/权限。

---

### B.5.2 Tool 注册失败

**现象：** MCP Server 启动了，但 Tool 没出现在可用列表中。

```
// claude mcp list 能看到 Server，但 Tool 不对
// 或者：Tool "my_awesome_tool" not found
```

**原因：**
- Server 的 `list_tools` 方法没有返回该 Tool
- Tool 的 schema 格式有误
- Server 初始化失败，静默跳过了 Tool 注册

**解决方案：**

```javascript
// MCP Server 端 —— 确保 Tool 正确注册
import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';

const server = new Server(
  { name: 'my-tools', version: '1.0.0' },
  { capabilities: { tools: {} } }  // ⚠️ 必须声明 tools 能力！
);

// 注册 Tool —— schema 必须符合 JSON Schema 规范
server.setRequestHandler('tools/list', async () => ({
  tools: [
    {
      name: 'my_awesome_tool',           // 名称只能用小写字母、数字、下划线
      description: '做了一些很酷的事情',   // 描述不能为空！
      inputSchema: {                       // ⚠️ 是 inputSchema，不是 input_schema
        type: 'object',
        properties: {
          query: {
            type: 'string',
            description: '搜索关键词',     // 每个字段都要有 description
          },
        },
        required: ['query'],
      },
    },
  ],
}));

// 实现 Tool 逻辑
server.setRequestHandler('tools/call', async (request) => {
  if (request.params.name === 'my_awesome_tool') {
    const { query } = request.params.arguments;
    return {
      content: [{
        type: 'text',
        text: `搜索结果：${query} 的相关内容...`,
      }],
    };
  }
  throw new Error(`未知 Tool: ${request.params.name}`);
});

// 启动
const transport = new StdioServerTransport();
await server.connect(transport);
```

**常见坑清单：**

| 错误 | 正确 |
|------|------|
| `input_schema` | `inputSchema` |
| Tool 名含大写/横杠 | 只用小写+下划线 |
| description 为空 | 必须有描述 |
| 忘了声明 `capabilities: { tools: {} }` | 必须声明 |

**老三名言：** MCP Tool 注册失败，99% 是 schema 写错了。剩下 1% 是忘了声明 capabilities。

---

### B.5.3 MCP Server 启动错误

**现象：** MCP Server 启动时崩溃或无响应。

```
Error: MCP server "my-tools" exited immediately with code 1
```

或：Server 启动后无任何输出，SDK 连接超时。

**原因：**
- 依赖缺失
- 端口冲突（SSE 模式）
- Node.js 版本不兼容
- 代码语法错误

**解决方案：**

```bash
# 1. 独立启动 Server 看错误信息
npx @my-org/mcp-server
# 或直接运行源码
cd /path/to/mcp-server && node index.js

# 2. 检查 Node.js 版本
node --version
# MCP SDK 通常需要 Node.js >= 18

# 3. 检查依赖
cd /path/to/mcp-server && npm install && npm ls

# 4. 端口冲突排查（SSE 模式）
lsof -i :3001  # 检查端口是否被占用
kill -9 <PID>  # 如果需要杀掉占用进程
```

```javascript
// 5. 添加启动日志，帮助调试
// 在 MCP Server 入口添加
process.on('uncaughtException', (error) => {
  console.error('[MCP Server] 未捕获异常:', error);
});

process.on('unhandledRejection', (reason) => {
  console.error('[MCP Server] 未处理的 Promise 拒绝:', reason);
});

// stdio 模式的 Server，日志不能输出到 stdout（会干扰 JSON-RPC 通信）
// 用 stderr 输出日志
console.error('[MCP Server] 启动中...');  // ✅ 用 stderr
// console.log('[MCP Server] 启动中...'); // ❌ 这会破坏 stdio 通信！
```

**老三血泪教训：** stdio 模式的 MCP Server，千万别用 `console.log` 打日志！那会直接污染 JSON-RPC 通道，导致通信全部乱套。用 `console.error` 或者写文件。

---

## B.6 调试技巧

### B.6.1 如何开启 debug 日志

**现象：** SDK 行为不符合预期，想看内部到底在干什么。

**解决方案：**

```bash
# 方法1：环境变量（最简单）
export CLAUDE_CODE_DEBUG=1          # 开启 SDK debug 模式
export ANTHROPIC_LOG=debug         # 开启 API 请求 debug 日志

claude query "你好"
# 现在你会看到大量内部日志

# 方法2：只在当前命令开启
CLAUDE_CODE_DEBUG=1 claude query "你好"
```

```javascript
// 方法3：SDK 代码中开启
import { ClaudeCode } from '@anthropic-ai/claude-code';

const sdk = new ClaudeCode({
  debug: true,  // 开启 debug 模式
  // 或者更细粒度
  logger: {
    debug: (...args) => console.error('[DEBUG]', ...args),
    info: (...args) => console.error('[INFO]', ...args),
    warn: (...args) => console.error('[WARN]', ...args),
    error: (...args) => console.error('[ERROR]', ...args),
  },
});
```

```bash
# 方法4：查看 MCP 通信日志
export MCP_DEBUG=1
claude query "用 my-tool 做点什么"
# 会打印 MCP Server 的完整通信内容
```

**老三建议：** debug 日志就像 X 光 —— 平时别开（太吵），出问题时一照就清楚。

---

### B.6.2 如何查看完整请求响应

**现象：** 想看 SDK 发给 API 的完整请求和收到的完整响应。

**解决方案：**

```bash
# 方法1：使用 ANTHROPIC_LOG 查看原始请求
export ANTHROPIC_LOG=debug
claude query "写个 hello world"
# 输出中会包含完整的 HTTP 请求和响应
```

```javascript
// 方法2：用中间件拦截请求和响应
import { ClaudeCode } from '@anthropic-ai/claude-code';

const sdk = new ClaudeCode({
  // 请求拦截
  onRequest: (request) => {
    console.error('=== 请求 ===');
    console.error('URL:', request.url);
    console.error('Model:', request.body?.model);
    console.error('Messages:', JSON.stringify(request.body?.messages, null, 2));
    console.error('Tools:', request.body?.tools?.map(t => t.name));
    console.error('估算 tokens:', request.body?.messages?.length, '条消息');
  },
  
  // 响应拦截
  onResponse: (response) => {
    console.error('=== 响应 ===');
    console.error('Status:', response.status);
    console.error('Model:', response.headers?.get('model'));
    console.error('Usage:', JSON.stringify(response.usage));
    console.error('Stop reason:', response.stop_reason);
  },
});
```

```bash
# 方法3：用代理工具抓包（高级）
# 安装 mitmproxy
pip install mitmproxy

# 启动代理
mitmproxy -p 8080

# 设置 SDK 走代理
export HTTPS_PROXY=http://localhost:8080
export NODE_TLS_REJECT_UNAUTHORIZED=0  # mitmproxy 用自签名证书

# 然后正常运行，在 mitmproxy 界面查看完整请求响应
```

**老三工具箱：** 日常调试用 `ANTHROPIC_LOG=debug`，复杂问题上 mitmproxy，够用了。

---

### B.6.3 如何用 curl 测试 API

**现象：** 想绕过 SDK，直接用 curl 测试 Anthropic API 是否正常。

**解决方案：**

```bash
# 基础请求 —— 测试 API 连通性
curl https://api.anthropic.com/v1/messages \
  -H "Content-Type: application/json" \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -d '{
    "model": "claude-sonnet-4-20250514",
    "max_tokens": 1024,
    "messages": [
      {"role": "user", "content": "说你好"}
    ]
  }'

# 带 Tool 的请求 —— 测试 Tool Use
curl https://api.anthropic.com/v1/messages \
  -H "Content-Type: application/json" \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -d '{
    "model": "claude-sonnet-4-20250514",
    "max_tokens": 4096,
    "messages": [
      {"role": "user", "content": "北京现在几度？"}
    ],
    "tools": [
      {
        "name": "get_weather",
        "description": "获取指定城市的天气",
        "input_schema": {
          "type": "object",
          "properties": {
            "city": {"type": "string", "description": "城市名"}
          },
          "required": ["city"]
        }
      }
    ]
  }'

# 流式请求 —— 测试 SSE
curl -N https://api.anthropic.com/v1/messages \
  -H "Content-Type: application/json" \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -d '{
    "model": "claude-sonnet-4-20250514",
    "max_tokens": 1024,
    "stream": true,
    "messages": [
      {"role": "user", "content": "说你好"}
    ]
  }'

# 一键检查 API Key 有效性
curl -s https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "claude-sonnet-4-20250514",
    "max_tokens": 10,
    "messages": [{"role": "user", "content": "hi"}]
  }' | python3 -c "import sys,json; r=json.load(sys.stdin); print('✅ Key 有效' if 'content' in r else '❌ Key 无效: ' + str(r))"
```

**老三终极建议：** curl 是调试 API 的瑞士军刀。SDK 出问题时，先用 curl 确认 API 本身没问题，再回来看 SDK 配置。分治法，永远的神。

---

> 🎯 **老三的 FAQ 总结：** 遇到问题别慌，按这个顺序排查：
> 1. 先看报错信息（认真读！）
> 2. 开 debug 日志看内部细节
> 3. 用 curl 测试 API 本身
> 4. 检查配置和环境变量
> 5. 还不行？去 [GitHub Issues](https://github.com/anthropics/claude-code/issues) 搜一搜
>
> _祝你好运，少踩坑，多摸鱼。_ 🐟
