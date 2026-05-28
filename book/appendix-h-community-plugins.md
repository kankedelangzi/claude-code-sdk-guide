# 附录H：社区插件与工具生态

> Claude Code SDK 不仅仅是一个独立的库，它拥有一个不断成长的插件体系和工具生态。本附录介绍官方插件、社区工具、IDE 扩展以及第三方集成，帮助你快速找到适合自己项目的工具。

## 目录

1. [生态概览](#生态概览)
2. [官方插件详解](#官方插件详解)
3. [SDK 插件系统](#sdk-插件系统)
4. [IDE 与编辑器扩展](#ide-与编辑器扩展)
5. [社区工具与框架](#社区工具与框架)
6. [MCP 服务器生态](#mcp-服务器生态)
7. [推荐工具清单](#推荐工具清单)
8. [参与社区贡献](#参与社区贡献)

---

## 生态概览

Claude Code 的生态由以下几层组成：

```
┌─────────────────────────────────────────────┐
│           应用层（你的产品）                  │
├─────────────────────────────────────────────┤
│  Agent SDK（核心编程接口）                   │
├─────────────────────────────────────────────┤
│  插件层（Skills / Agents / Hooks / MCP）    │
├─────────────────────────────────────────────┤
│  工具层（IDE 扩展 / CLI 工具 / 监控）        │
├─────────────────────────────────────────────┤
│  社区层（Discord / GitHub / 插件市场）       │
└─────────────────────────────────────────────┘
```

### 生态资源导航

| 资源 | 地址 | 用途 |
|------|------|------|
| 官方文档 | code.claude.com/docs | 完整文档 |
| GitHub 仓库 | github.com/anthropics/claude-code | 源码、Issue、PR |
| Discord 社区 | anthropic.com/discord | 问答、交流 |
| 插件市场 | code.claude.com/docs/en/plugin-marketplaces | 第三方插件 |
| llms.txt | code.claude.com/docs/llms.txt | LLM 友好的文档索引 |

---

## 官方插件详解

Claude Code 官方在仓库的 `plugins/` 目录提供了一系列高质量插件，可以直接使用或作为开发参考。

### 插件安装方式

官方插件以目录形式存放在仓库中，使用时需要克隆仓库并指定本地路径：

```bash
# 克隆官方仓库
git clone https://github.com/anthropics/claude-code.git

# 在 SDK 中引用插件（见后续章节）
```

### agent-sdk-dev

**功能**：Agent SDK 应用开发工具包

- **命令**：`/new-sdk-app` — 交互式创建新的 Agent SDK 项目
- **Agent**：`agent-sdk-verifier-py`、`agent-sdk-verifier-ts` — 验证 SDK 应用是否符合最佳实践

**使用场景**：从头创建 SDK 项目，或审查现有项目的架构质量。

```typescript
// 在 SDK 中加载此插件
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "Create a new SDK app for code review",
  options: {
    plugins: [
      { type: "local", path: "./claude-code/plugins/agent-sdk-dev" }
    ]
  }
})) {
  console.log(message);
}
```

### code-review

**功能**：自动化 PR 代码审查

- **命令**：`/code-review` — 触发自动化 PR 审查流程
- **Agent**：5 个并行 Sonnet Agent：
  - CLAUDE.md 合规性检查
  - Bug 检测
  - 历史上下文分析
  - PR 历史比对
  - 代码注释质量

**核心特点**：基于置信度评分过滤误报，减少噪音。

```typescript
// 在 CI 中使用 code-review 插件
import { query, ClaudeAgentOptions } from "@anthropic-ai/claude-agent-sdk";

async function reviewPR(prNumber: number) {
  const messages = [];
  for await (const msg of query({
    prompt: `/code-review PR #${prNumber}`,
    options: {
      plugins: [
        { type: "local", path: "./plugins/code-review" }
      ]
    }
  })) {
    messages.push(msg);
  }
  return messages;
}
```

### feature-dev

**功能**：结构化功能开发工作流

- **命令**：`/feature-dev` — 引导式功能开发
- **7 阶段流程**：需求分析 → 代码探索 → 架构设计 → 实现 → 测试 → 审查 → 文档
- **Agent**：`code-explorer`、`code-architect`、`code-reviewer`

**适用场景**：复杂功能的端到端开发，确保质量和一致性。

### commit-commands

**功能**：Git 工作流自动化

- **命令**：
  - `/commit` — 智能提交（自动生成 commit message）
  - `/commit-push-pr` — 提交 + 推送 + 创建 PR 一站式
  - `/clean_gone` — 清理已合并的本地分支

```bash
# 在 Claude Code CLI 中使用
claude> /commit
# Claude 会自动分析 git diff，生成规范的 commit message

claude> /commit-push-pr
# 自动完成：commit → push → 创建 PR（含描述）
```

### frontend-design

**功能**：生产级前端界面设计

- **Skill**：`frontend-design` — 自动调用，避免通用 AI 美学
- **设计指导**：大胆的设计选择、排版、动画、视觉细节

**使用场景**：需要高质量 UI 设计时，避免"AI 默认风格"。

### hookify

**功能**：创建自定义 Hook 防止不良行为

- **命令**：
  - `/hookify` — 分析对话，自动生成 Hook 规则
  - `/hookify:list` — 列出所有 Hook
  - `/hookify:configure` — 配置 Hook 参数
- **Agent**：`conversation-analyzer` — 分析对话模式

```typescript
// hookify 生成的 Hook 示例（自动防止某些行为）
// .claude/settings.json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "command",
        "command": "bash -c 'echo \"$TOOL_INPUT\" | grep -q \"rm -rf\" && exit 1 || exit 0'"
      }]
    }]
  }
}
```

### claude-opus-4-5-migration

**功能**：从 Sonnet 4.x / Opus 4.1 迁移到 Opus 4.5

- **Skill**：`claude-opus-4-5-migration` — 自动迁移
- **迁移内容**：模型字符串、beta headers、prompt 调整

### 其他官方插件

| 插件名 | 功能 |
|--------|------|
| `explanatory-output-style` | 在教育模式下添加实现说明（通过 Hook 实现） |
| `learning-output-style` | 交互式学习模式 |

---

## SDK 插件系统

> Agent SDK 支持通过编程方式加载插件，这是扩展 SDK 功能的主要方式。

### 插件结构

一个标准插件目录结构如下：

```
my-plugin/
├── .claude-plugin/
│   └── plugin.json          # 插件元数据
├── skills/                  # 技能定义（推荐）
│   └── my-skill.md
├── agents/                  # 专用 Agent 定义
│   └── my-agent.md
├── hooks/                   # Hook 脚本
│   └── my-hook.sh
├── mcp.json                 # MCP 服务器配置
└── commands/                # 旧格式命令（已弃用，用 skills 替代）
    └── my-command.md
```

### plugin.json 格式

```json
{
  "name": "my-plugin",
  "version": "1.0.0",
  "description": "My custom Claude Code plugin",
  "author": "Your Name",
  "entryPoint": "skills/my-skill.md"
}
```

### 在 SDK 中加载插件

```typescript
import { query, ClaudeAgentOptions } from "@anthropic-ai/claude-agent-sdk";
import * as path from "path";

// 加载单个插件
const pluginPath = path.resolve("./plugins/my-plugin");

for await (const message of query({
  prompt: "Use the custom skill from my plugin",
  options: {
    plugins: [
      { type: "local", path: pluginPath }
    ]
  }
})) {
  if (message.type === "text") {
    console.log(message.text);
  }
}
```

### 加载多个插件

```typescript
// 同时加载多个插件
for await (const message of query({
  prompt: "Run code review with custom hooks",
  options: {
    plugins: [
      { type: "local", path: "./plugins/code-review" },
      { type: "local", path: "./plugins/my-custom-hooks" },
      { type: "local", path: "/absolute/path/to/shared-plugin" }
    ]
  }
})) {
  // 所有插件的功能同时可用
}
```

### 验证插件加载

插件加载成功后，会出现在系统初始化消息中。可以通过监听 `system` 类型消息来验证：

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

async function checkPlugins() {
  for await (const message of query({
    prompt: "Hello",
    options: {
      plugins: [{ type: "local", path: "./plugins/code-review" }]
    }
  })) {
    if (message.type === "system") {
      console.log("Loaded plugins:", message.data?.plugins);
      // 输出: ["code-review", ...]
    }
  }
}
```

---

## IDE 与编辑器扩展

### Visual Studio Code 扩展

Claude Code 提供了官方的 VSCode 扩展，支持：

- 侧边栏聊天面板
- 内联代码补全
- 选中代码解释 / 重构
- 集成终端

**安装方式**：

```
# VSCode 扩展市场搜索
Claude Code

# 或直接从 VSIX 安装（如果提供）
code --install-extension claude-code.vsix
```

**配置示例**（VSCode settings.json）：

```json
{
  "claudeCode.apiKey": "your-api-key",
  "claudeCode.model": "claude-sonnet-4-20250514",
  "claudeCode.enableInlineCompletion": true,
  "claudeCode.autoImport": true
}
```

### JetBrains IDEs（IntelliJ / WebStorm / PyCharm）

通过 Marketplace 安装 Claude Code 插件：

1. 打开 IDE → Preferences → Plugins
2. 搜索 "Claude Code"
3. 安装并重启

**主要功能**：
- 侧边栏对话
- 代码选中后右键 "Ask Claude"
- 自动补全建议

### Neovim / Vim

社区提供了 Neovim 集成插件（非官方）：

```lua
-- 使用 lazy.nvim 安装
{
  "anthropics/claude-code.nvim",
  config = function()
    require("claude-code").setup({
      api_key = os.getenv("ANTHROPIC_API_KEY"),
      model = "claude-sonnet-4-20250514"
    })
  end
}
```

**常用快捷键**：

| 快捷键 | 功能 |
|--------|------|
| `<leader>cc` | 打开 Claude Code 面板 |
| `<leader>ce` | 解释选中代码 |
| `<leader>cr` | 重构选中代码 |

### Chrome / Firefox 扩展

官方提供了浏览器扩展，可以在网页中使用 Claude Code：

- 选中网页文本 → 右键 "Ask Claude"
- GitHub PR 页面自动代码审查
- 任意文本框 "AI 辅助撰写"

**安装**：Chrome Web Store 搜索 "Claude Code"

---

## 社区工具与框架

### 1. OpenClaw

**简介**：跨平台 AI 助手框架，支持 Claude Code SDK 作为后端

**核心功能**：
- 多 Channel 支持（Discord、Telegram、WhatsApp）
- 技能（Skill）系统，类似 Claude Code 插件
- 定时任务（Cron）
- 子 Agent 编排

**安装**：

```bash
# 安装 OpenClaw
npm install -g openclaw

# 配置 Claude Code SDK 作为后端
openclaw config set model qclaw/modelroute
openclaw config set provider anthropic
```

**与 Claude Code SDK 的关系**：

OpenClaw 使用 Claude Code SDK（Agent SDK）作为底层 AI 引擎，在其上构建了多通道交互层。

```typescript
// OpenClaw 内部使用 Agent SDK 的方式（概念性）
import { query } from "@anthropic-ai/claude-agent-sdk";

async function handleUserMessage(userInput: string) {
  for await (const message of query({
    prompt: userInput,
    options: {
      // OpenClaw 注入系统 prompt 和 tools
      systemPrompt: buildSystemPrompt(),
      tools: registerTools()
    }
  })) {
    // 将 SDK 输出发送到 Discord/Telegram
    sendToChannel(message);
  }
}
```

### 2. LangChain Claude 集成

**简介**：LangChain 提供了 Claude 模型的支持

```python
# Python - LangChain 使用 Claude
from langchain_anthropic import ChatAnthropic

llm = ChatAnthropic(
    model="claude-sonnet-4-20250514",
    anthropic_api_key="your-api-key"
)

response = llm.invoke("Explain this code...")
```

**与 Agent SDK 的对比**：

| 特性 | Agent SDK | LangChain |
|------|-----------|-----------|
| 原生支持 | ✅ 官方 | ✅ 通过集成 |
| Tools 定义 | 简洁，原生 | 需要通过 LangChain Tool |
| 多 Agent | ✅ 原生支持 | 需要 LangGraph |
| Claude 特定功能 | ✅ 完整 | 部分 |

### 3. Vercel AI SDK

**简介**：Vercel 的 AI SDK 支持 Claude 模型

```typescript
// Next.js + Vercel AI SDK + Claude
import { Anthropic } from "@ai-sdk/anthropic";
import { streamText } from "ai";

const anthropic = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY
});

const result = await streamText({
  model: anthropic("claude-sonnet-4-20250514"),
  prompt: "Write a function to..."
});
```

### 4. Promptfoo

**简介**：AI 提示词测试和评估框架，支持 Claude

```bash
# 安装
npm install -g promptfoo

# 配置 Claude 作为测试目标
promptfoo eval --config promptfoo.yaml
```

**promptfoo.yaml**：

```yaml
providers:
  - id: anthropic:claude-sonnet-4-20250514
    config:
      temperature: 0.7

prompts:
  - file://prompt-template.txt

tests:
  - description: "Code generation quality"
    vars:
      task: "Write a binary search function"
    assert:
      - type: contains
        value: "function binarySearch"
```

### 5. Helicone

**简介**：Claude API 的监控和缓存中间件

```typescript
// 使用 Helicone 包装 Anthropic 客户端
import { Anthropic } from "@anthropic-ai/sdk";
import { Helicone } from "@helicone/helicone";

const client = new Anthropic({
  apiKey: "your-key",
  baseURL: "https://anthropic.helicone.ai/v1",
  defaultHeaders: {
    "Helicone-Auth": "Bearer your-helicone-key"
  }
});
```

**功能**：
- 请求/响应日志
- 成本跟踪
- Prompt 缓存（无需改代码）
- 速率限制

---

## MCP 服务器生态

> Model Context Protocol (MCP) 是 Claude Code 连接外部工具的标准协议。社区已经构建了丰富的 MCP 服务器。

### 官方 MCP 服务器

Anthropic 提供了一些官方 MCP 服务器参考实现：

| 服务器 | 功能 | 地址 |
|--------|------|------|
| Filesystem | 文件系统访问 | github.com/modelcontextprotocol/servers |
| Database | PostgreSQL / SQLite 访问 | 同上 |
| Web Search | 网页搜索 | 同上 |
| Slack | Slack 消息收发 | 同上 |
| GitHub | GitHub 操作 | 同上 |

### 在 SDK 中连接 MCP 服务器

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "Search the web for recent AI news",
  options: {
    mcpServers: {
      "web-search": {
        command: "npx",
        args: ["-y", "@modelcontextprotocol/server-web-search"]
      }
    }
  }
})) {
  console.log(message);
}
```

### 社区 MCP 服务器精选

```typescript
// 社区热门 MCP 服务器配置示例
const mcpServers = {
  // Notion 集成
  "notion": {
    command: "npx",
    args: ["-y", "@notionhq/mcp-server-notion"],
    env: { NOTION_TOKEN: process.env.NOTION_TOKEN }
  },

  // Linear（项目管理）
  "linear": {
    command: "npx",
    args: ["-y", "@linear/mcp-server"],
    env: { LINEAR_API_KEY: process.env.LINEAR_API_KEY }
  },

  // Docker 操作
  "docker": {
    command: "docker",
    args: ["run", "-i", "--rm", "mcp/docker"]
  },

  // Kubernetes
  "kubernetes": {
    command: "npx",
    args: ["-y", "@modelcontextprotocol/server-kubernetes"]
  }
};
```

### 构建自己的 MCP 服务器

```typescript
// 一个简单的自定义 MCP 服务器
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { tools } from "@modelcontextprotocol/sdk/types.js";

const server = new Server(
  { name: "my-mcp-server", version: "1.0.0" },
  { capabilities: { tools: {} } }
);

// 注册工具
server.setRequestHandler(tools.listTools, async () => ({
  tools: [
    {
      name: "get_weather",
      description: "Get current weather for a city",
      inputSchema: {
        type: "object",
        properties: {
          city: { type: "string" }
        },
        required: ["city"]
      }
    }
  ]
}));

// 处理工具调用
server.setRequestHandler(tools.callTool, async (request) => {
  if (request.params.name === "get_weather") {
    const city = request.params.arguments.city;
    const weather = await fetchWeather(city);
    return { content: [{ type: "text", text: weather }] };
  }
  throw new Error("Unknown tool");
});

// 启动服务器
const transport = new StdioServerTransport();
await server.connect(transport);
```

---

## 推荐工具清单

### 开发工具

| 工具 | 类型 | 推荐场景 |
|------|------|----------|
| **OpenClaw** | 多通道 AI 框架 | 构建 Discord/Telegram Bot |
| **Promptfoo** | 测试评估 | Prompt 工程、A/B 测试 |
| **Helicone** | 监控中间件 | 生产环境监控、缓存 |
| **LangSmith** | 追踪平台 | 调试复杂 Agent 流程 |
| **Braintrust** | 评估平台 | 大规模 LLM 评估 |

### 代码质量

| 工具 | 功能 |
|------|------|
| **code-review 插件** | 自动化 PR 审查 |
| **feature-dev 插件** | 结构化功能开发 |
| **ESLint + Claude** | AI 辅助代码规范检查 |

### 部署与运维

| 工具 | 功能 |
|------|------|
| **Docker** | 容器化部署 |
| **Prometheus + Grafana** | 监控告警（见第25章） |
| **Helicone** | API 监控中间件 |
| **Vercel / Railway** | 快速部署平台 |

### 学习资源

| 资源 | 地址 | 类型 |
|------|------|------|
| 官方文档 | code.claude.com/docs | 文档 |
| Discord 社区 | anthropic.com/discord | 社区 |
| GitHub Discussions | github.com/anthropics/claude-code/discussions | Q&A |
| Anthropic Cookbook | github.com/anthropics/anthropic-cookbook | 示例 |
| Awesome Claude Code | github.com/搜索 | 社区汇总 |

---

## 参与社区贡献

### 贡献方式

1. **提交插件**：将你开发的插件提交到社区插件市场
2. **报告 Bug**：在 GitHub Issues 中报告问题
3. **改进文档**：提交 PR 改进官方文档
4. **回答问答**：在 Discord 帮助其他开发者

### 插件发布检查清单

在发布你的插件之前，确保：

- [ ] `plugin.json` 包含完整的元数据
- [ ] 所有 Agent/Skill 定义都有清晰的 `description`
- [ ] 提供了使用示例
- [ ] 添加了 `README.md`
- [ ] 遵循官方插件目录结构

### 插件 README.md 模板

```markdown
# My Plugin

> 简短描述你的插件功能

## 安装

\`\`\`bash
git clone https://github.com/yourname/my-plugin.git
\`\`\`

## 使用

\`\`\`typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "...,
  options: {
    plugins: [{ type: "local", path: "./my-plugin" }]
  }
})) {
  console.log(message);
}
\`\`\`

## 功能

- 功能1：...
- 功能2：...

## 要求

- Claude Agent SDK >= 1.0.0

## 许可

MIT
```

---

## 总结

Claude Code 的生态正在快速发展：

- **官方插件** 提供了高质量的功能扩展（code-review、feature-dev 等）
- **SDK 插件系统** 让你以标准化方式扩展功能
- **IDE 扩展** 将 Claude Code 能力带入日常开发
- **MCP 协议** 连接了丰富的外部工具生态
- **社区工具**（OpenClaw、Promptfoo、Helicone）填补了特定场景的需求

**推荐下一步**：
1. 尝试官方插件（特别是 code-review 和 feature-dev）
2. 学习 MCP 协议，连接自己的数据源
3. 加入 Discord 社区，获取最新动态

---

> **下一附录**：附录I 将介绍真实公司的 Claude Code SDK 实施案例，包括架构决策和经验教训。
