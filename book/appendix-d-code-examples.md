# 附录D：完整代码示例仓库说明

> 本书所有代码示例的配套仓库使用指南
> 仓库地址：https://github.com/your-username/claude-code-sdk-examples

## D.1 仓库概述

学编程，光看不练假把式。

本书每一章都有对应的可运行代码，全部放在这个仓库里。你可以：
- ✅ 直接克隆到本地运行
- ✅ 对照书本章节逐步调试
- ✅ 基于示例代码做修改和实验
- ✅ 遇到问题时参考 working code

**仓库结构一览：**

```
claude-code-sdk-examples/
├── 01-introduction/          # 第1章：简介（无代码）
├── 02-installation/          # 第2章：安装验证脚本
├── 03-first-program/         # 第3章：第一个 SDK 程序
├── 04-concepts/              # 第4章：核心概念演示
├── 05-message-content/       # 第5章：Message 与 Content 类型
├── 06-tool-definition/       # 第6章：Tool 定义与调用
├── 07-mcp-protocol/          # 第7章：MCP 协议示例
├── 08-claudemd-config/       # 第8章：CLAUDE.md 配置示例
├── 09-multi-turn/            # 第9章：多轮对话管理
├── 10-smart-assistant/       # 第10章：智能助手实战
├── 11-code-generator/        # 第11章：代码生成工具
├── 12-performance/           # 第12章：性能优化示例
├── 13-prompt-engineering/    # 第13章：高级 Prompt 工程
├── 14-multi-agent/           # 第14章：多 Agent 协作
├── 15-production/            # 第15章：生产环境部署
├── 16-plugin-dev/            # 第16章：插件开发指南
├── 17-debugging/             # 第17章：调试技巧
├── appendix-a-api/           # 附录A：API 参考示例
├── appendix-b-faq/           # 附录B：FAQ 代码示例
├── appendix-c-cli-vs-sdk/    # 附录C：CLI vs SDK 对比
└── shared/                   # 共享工具和配置
    ├── utils.js              # 通用工具函数
    ├── config.example.js     # 配置模板
    └── package.json          # 依赖管理
```

---

## D.2 快速开始

### D.2.1 克隆仓库

```bash
# HTTPS 方式（推荐）
git clone https://github.com/your-username/claude-code-sdk-examples.git

# SSH 方式（如果你配置了 SSH key）
git clone git@github.com:your-username/claude-code-sdk-examples.git

# 进入目录
cd claude-code-sdk-examples
```

### D.2.2 安装依赖

仓库根目录有一个统一的 `package.json`，包含所有章节的公共依赖：

```bash
# 安装所有依赖
npm install

# 或者如果你只用某些章节，可以按需安装
cd 03-first-program
npm install
```

**公共依赖（根目录 package.json）：**

```json
{
  "name": "claude-code-sdk-examples",
  "version": "1.0.0",
  "description": "《Claude Code SDK 编程指南》配套代码示例",
  "scripts": {
    "test": "echo \"Run examples with: node <chapter>/index.js\"",
    "test:all": "npm run test --workspaces",
    "lint": "eslint . --ext .js,.mjs",
    "format": "prettier --write \"**/*.js\""
  },
  "dependencies": {
    "@anthropic-ai/claude-code-sdk": "^1.0.0",
    "dotenv": "^16.4.0"
  },
  "devDependencies": {
    "eslint": "^8.57.0",
    "prettier": "^3.3.0"
  },
  "workspaces": [
    "01-introduction",
    "02-installation",
    "03-first-program",
    "04-concepts",
    "05-message-content",
    "06-tool-definition",
    "07-mcp-protocol",
    "08-claudemd-config",
    "09-multi-turn",
    "10-smart-assistant",
    "11-code-generator",
    "12-performance",
    "13-prompt-engineering",
    "14-multi-agent",
    "15-production",
    "16-plugin-dev",
    "17-debugging"
  ]
}
```

### D.2.3 配置 API Key

所有需要调用 Claude API 的示例都依赖 `ANTHROPIC_API_KEY` 环境变量。

**方法一：.env 文件（推荐）**

```bash
# 复制配置模板
cp shared/.env.example .env

# 编辑 .env，填入你的 API Key
# ANTHROPIC_API_KEY=sk-ant-...

# 也支持自定义 Base URL（如果你用代理）
# ANTHROPIC_BASE_URL=https://api.anthropic.com
```

**.env.example 模板：**

```bash
# Claude API 配置
ANTHROPIC_API_KEY=your_api_key_here
ANTHROPIC_BASE_URL=https://api.anthropic.com

# 可选：模型选择（默认 claude-sonnet-4-20250514）
# MODEL=claude-opus-4-20250514

# 可选：日志级别（debug | info | warn | error）
# LOG_LEVEL=info
```

**方法二：环境变量（适合生产环境）**

```bash
# Linux / macOS
export ANTHROPIC_API_KEY="sk-ant-..."

# Windows (PowerShell)
$env:ANTHROPIC_API_KEY="sk-ant-..."

# Windows (CMD)
set ANTHROPIC_API_KEY=sk-ant-...
```

### D.2.4 运行第一个示例

```bash
# 进入第3章示例目录
cd 03-first-program

# 运行第一个 SDK 程序
node index.js
```

**预期输出：**

```
🚀 Claude Code SDK 第一个程序
✅ API Key 已配置
✅ SDK 版本: 1.0.0
📝 发送消息: "用一句话介绍 Claude"

Claude 回复：
  Claude 是 Anthropic 公司开发的人工智能助手，
  致力于提供有用、无害、诚实的对话体验。

⏱️  耗时: 1234 ms
💰 消耗 Token: 输入 15, 输出 42
```

---

## D.3 分章节使用指南

### D.3.1 第2章：环境搭建与安装

**目录：** `02-installation/`

**示例列表：**

| 文件 | 说明 | 运行命令 |
|------|------|---------|
| `verify-install.js` | 验证 SDK 安装是否成功 | `node verify-install.js` |
| `check-version.js` | 检查 SDK 版本 | `node check-version.js` |
| `test-connection.js` | 测试 API 连接 | `node test-connection.js` |

**代码示例（verify-install.js）：**

```javascript
// 02-installation/verify-install.js
import { SDK_VERSION } from '@anthropic-ai/claude-code-sdk';

console.log('✅ Claude Code SDK 安装验证');
console.log('📦 SDK 版本:', SDK_VERSION);

// 检查环境变量
if (!process.env.ANTHROPIC_API_KEY) {
  console.warn('⚠️  未检测到 ANTHROPIC_API_KEY 环境变量');
  console.warn('   请创建 .env 文件或设置环境变量');
  process.exit(1);
}

console.log('✅ API Key 已配置');
console.log('🎉 环境验证通过，可以开始编程！');
```

---

### D.3.2 第3章：第一个 SDK 程序

**目录：** `03-first-program/`

**示例列表：**

| 文件 | 说明 | 运行命令 |
|------|------|---------|
| `index.js` | 最简单的 SDK 调用 | `node index.js` |
| `with-stream.js` | 流式输出版本 | `node with-stream.js` |
| `with-system-prompt.js` | 带系统提示词 | `node with-system-prompt.js` |
| `error-handling.js` | 错误处理示例 | `node error-handling.js` |

**代码示例（index.js）：**

```javascript
// 03-first-program/index.js
import { query } from '@anthropic-ai/claude-code-sdk';

async function main() {
  console.log('🚀 Claude Code SDK 第一个程序\n');

  const startTime = Date.now();

  const messages = [];
  for await (const event of query({
    model: 'claude-sonnet-4-20250514',
    prompt: '用一句话介绍 Claude',
    maxTurns: 1,
  })) {
    if (event.type === 'assistant') {
      messages.push(event.message.content[0].text);
    }
  }

  const duration = Date.now() - startTime;

  console.log('Claude 回复：');
  console.log('  ' + messages.join(''));
  console.log(`\n⏱️  耗时: ${duration} ms`);
}

main().catch(console.error);
```

---

### D.3.3 第5章：Message 与 Content 类型

**目录：** `05-message-content/`

**示例列表：**

| 文件 | 说明 | 运行命令 |
|------|------|---------|
| `text-message.js` | 纯文本消息 | `node text-message.js` |
| `image-message.js` | 图片消息（Base64） | `node image-message.js` |
| `tool-result-message.js` | 工具结果消息 | `node tool-result-message.js` |
| `conversation-history.js` | 对话历史管理 | `node conversation-history.js` |

**代码示例（image-message.js）：**

```javascript
// 05-message-content/image-message.js
import { query } from '@anthropic-ai/claude-code-sdk';
import fs from 'fs';
import path from 'path';

async function main() {
  // 读取图片并转为 Base64
  const imagePath = path.join(process.cwd(), 'example-image.png');
  const imageBuffer = fs.readFileSync(imagePath);
  const base64Image = imageBuffer.toString('base64');

  console.log('🖼️  发送图片给 Claude...\n');

  const messages = [];
  for await (const event of query({
    model: 'claude-sonnet-4-20250514',
