# 第1章：Claude Code SDK 简介

## 1.1 什么是 Claude Code SDK

Claude Code SDK 是由 Anthropic 公司推出的**代理式（Agentic）编程工具包**，它将强大的 Claude AI 能力直接集成到开发者的工作流程中。与传统的代码补全工具不同，Claude Code SDK 是一个具备**记忆、工具调用、自主规划和环境感知能力**的智能代理系统。

### 核心定位

Claude Code SDK 不是简单的代码生成工具，而是：

- **全流程编程 Agent**：能够理解需求、规划步骤、执行任务、验证结果
- **系统级代理**：拥有文件读写、命令执行、Git 操作等系统级权限
- **上下文感知**：能够深度理解整个代码库的结构和依赖关系

### 官方定义

According to Anthropic 官方文档，Claude Code SDK 支持将 Claude Code 作为子进程运行，提供了一种构建 AI 驱动的编码助手和工具的方法。

```bash
# Claude Code SDK 的基本形式
claude -p "编写一个计算斐波那契数的函数"
```

---

## 1.2 SDK 的核心能力

Claude Code SDK 的核心能力可以归纳为以下几个维度：

### 1.2.1 深度代码库理解

Claude Code SDK 能够快速分析和理解大型代码库的结构，包括：

- **多文件关联分析**：自动遍历整个代码库，理解文件之间的依赖关系
- **上下文保持**：通过 CLAUDE.md 配置文件实现长期记忆
- **动态探索**：像开发者一样通过 grep、ls 等命令动态探索代码库

**示例：理解新代码库**

```bash
# 启动 Claude Code
cd your-project
claude

# 向 Claude Code 提问
> 这个代码库的整体架构是什么？
> 主要的入口文件在哪里？
> 这个功能模块是如何工作的？
```

Claude Code 会自动探索你的代码库，读取相关文件，并给出详细的分析。

### 1.2.2 多文件编辑能力

与传统 AI 补全工具只修改当前文件不同，Claude Code SDK 支持：

- **跨文件修改**：修改一个函数时，自动更新相关的测试、文档和配置文件
- **批量重构**：一次性重构多个文件中的相似代码
- **依赖更新**：自动更新受影响的导入语句和类型定义

**示例：跨文件重构**

```typescript
// 假设你要重命名一个在函数多个文件中使用的 API 函数

// 你可以这样向 Claude Code 描述需求：
"> 将所有的 fetchUserData() 函数重命名为 getUserProfile()，
> 并更新所有调用该函数的地方，包括测试文件和文档"
```

Claude Code 会自动：
1. 搜索所有调用 `fetchUserData()` 的文件
2. 重命名函数定义
3. 更新所有调用点
4. 更新相关测试
5. 更新文档中的示例代码

### 1.2.3 工具调用框架

Claude Code SDK 内置了强大的工具系统，包括：

| 工具类别 | 工具名称 | 功能描述 |
|---------|---------|---------|
| 文件操作 | Read | 读取文件内容 |
| 文件操作 | Write | 写入文件 |
| 文件操作 | Edit | 编辑文件（精确替换） |
| Shell 执行 | Bash | 执行任意 shell 命令 |
| 代码搜索 | Grep | 搜索代码内容 |
| 文件查找 | Glob | 查找文件路径 |
| Git 操作 | Git | 执行 git 命令 |
| 记忆系统 | TodoWrite | 管理任务列表 |

**示例：使用工具调用**

```typescript
// 使用 Claude Code SDK 的 TypeScript API
import { ClaudeCode } from '@anthropic-ai/claude-code-sdk';

const claude = new ClaudeCode();

// Claude 会自动调用工具来完成任务
const result = await claude.run({
  prompt: "找出所有包含 TODO 注释的文件，并生成一个汇总报告",
  tools: ['Read', 'Grep', 'Write']
});

console.log(result);
```

### 1.2.4 Agentic 智能体机制

Claude Code SDK 采用**代理式（Agentic）架构**，这意味着：

1. **自主规划**：Claude 能够将复杂任务分解为多个步骤
2. **工具调用**：在需要时主动调用合适的工具
3. **结果验证**：检查执行结果，必要时进行重试或调整
4. **多轮对话**：通过多轮交互逐步完成任务

**架构示意图：**

```
用户输入
   ↓
Claude Code SDK (主循环)
   ↓
┌─────────────────────────────────────┐
│  思考（Think）                      │
│  - 理解需求                         │
│  - 制定计划                         │
└─────────────────────────────────────┘
   ↓
┌─────────────────────────────────────┐
│  调用工具（Tool Use）               │
│  - Read 文件                        │
│  - Write 文件                       │
│  - Bash 命令                        │
│  - Grep 搜索                        │
└─────────────────────────────────────┘
   ↓
┌─────────────────────────────────────┐
│  观察结果（Observe）                │
│  - 检查工具执行结果                 │
│  - 判断是否需要继续                 │
└─────────────────────────────────────┘
   ↓
完成任务 / 继续下一轮
```

---

## 1.3 适用场景

Claude Code SDK 适用于多种开发场景，特别是那些需要深度理解代码库和执行复杂任务的场景。

### 1.3.1 快速原型开发

**场景描述**：你需要快速搭建一个项目原型，验证一个想法。

**Claude Code 的优势**：
- 自动生成项目骨架
- 编写基础功能代码
- 生成配套的测试代码

**示例：**

```bash
claude -p "创建一个 React + TypeScript 的待办事项应用，
包含添加、删除、标记完成功能，
使用 localStorage 持久化数据"
```

Claude Code 会自动：
1. 创建项目结构
2. 编写组件代码
3. 添加样式
4. 生成测试用例
5. 更新 package.json

### 1.3.2 Bug 修复与代码审查

**场景描述**：你发现了一个 bug，但不确定根因和修复方案。

**Claude Code 的优势**：
- 自动分析错误日志
- 定位问题代码
- 提出修复方案
- 生成测试用例验证修复

**示例：**

```bash
claude -p "修复 issue #123 中描述的空指针异常，
先分析根因，然后提供修复方案，最后编写测试验证"
```

### 1.3.3 大规模代码重构

**场景描述**：你需要重构一个大型项目，但担心引入回归 bug。

**Claude Code 的优势**：
- 理解整个代码库的依赖关系
- 自动更新所有受影响的地方
- 生成迁移脚本和文档

**示例：**

```bash
claude -p "将项目从 JavaScript 迁移到 TypeScript，
先迁移工具函数，然后是组件，最后是测试用例"
```

### 1.3.4 测试用例自动生成

**场景描述**：你的项目缺少测试覆盖，需要补充测试用例。

**Claude Code 的优势**：
- 分析现有代码逻辑
- 生成全面的测试用例
- 包括边界条件和异常处理

**示例：**

```bash
claude -p "为 src/utils/ 目录下的所有函数生成测试用例，
使用 Jest 框架，覆盖正常情况、边界情况和异常情况"
```

### 1.3.5 Git 工作流自动化

**场景描述**：你需要规范化 Git 提交信息，或者自动化 PR 流程。

**Claude Code 的优势**：
- 自动生成规范的提交信息
- 创建 Pull Request
- 解决合并冲突

**示例：**

```bash
claude -p "将我的修改分成3个有意义的提交，
每个提交都有规范的提交信息，
然后创建 PR 到 main 分支"
```

---

## 1.4 与其他工具的比较

为了更好地理解 Claude Code SDK 的定位，我们将其与其他流行的 AI 编程工具进行比较。

### 1.4.1 Claude Code vs GitHub Copilot

| 特性 | Claude Code SDK | GitHub Copilot |
|-----|----------------|---------------|
| **工作方式** | 代理式（自主规划和执行） | 补全式（提供建议） |
| **代码库理解** | 深度理解（可读取整个代码库） | 有限理解（主要基于当前文件） |
| **文件操作** | 支持多文件读写 | 只修改当前文件 |
| **命令执行** | 可以执行 shell 命令 | 不能执行命令 |
| **Git 集成** | 深度集成（自动提交、创建 PR） | 无 Git 集成 |
| **适用场景** | 复杂任务、多文件重构 | 快速补全、简单修改 |

**总结**：
- **GitHub Copilot** 适合快速代码补全和简单修改
- **Claude Code SDK** 适合复杂任务、多文件操作和自主执行

### 1.4.2 Claude Code vs Cursor

| 特性 | Claude Code SDK | Cursor |
|-----|----------------|--------|
| **界面** | 命令行 + IDE 插件 | IDE（基于 VSCode） |
| **工作方式** | 代理式 | 补全 + 对话式 |
| **代码库索引** | 动态探索（无索引） | 预先索引 |
| **定制化** | 高度可定制（MCP、Skills） | 中等定制 |
| **适用人群** | 命令行爱好者、高级开发者 | IDE 用户 |

**总结**：
- **Cursor** 适合喜欢 IDE 界面和实时补全的开发者
- **Claude Code SDK** 适合喜欢命令行和完全控制权的开发者

### 1.4.3 Claude Code vs Aider

| 特性 | Claude Code SDK | Aider |
|-----|----------------|-------|
| **模型支持** | 仅 Claude | 多种模型（GPT、Claude、本地模型） |
| **Git 集成** | 深度集成 | 深度集成 |
| **代码库理解** | 更强（支持动态探索） | 较强 |
| **工具生态** | 丰富（MCP、Skills） | 有限 |
| **成熟度** | 较新（2025 推出） | 较成熟 |

**总结**：
- **Aider** 适合需要多模型支持或本地模型的开发者
- **Claude Code SDK** 适合深度使用 Claude 生态的开发者

---

## 1.5 技术架构概述

理解 Claude Code SDK 的技术架构，有助于更好地使用和扩展它。

### 1.5.1 整体架构

Claude Code SDK 的架构可以分为四个主要层次：

```
┌─────────────────────────────────────┐
│   交互层（Interaction Layer）       │
│   - CLI 命令行                      │
│   - IDE 插件（VS Code、JetBrains） │
│   - Desktop App                     │
│   - Web 界面                        │
└─────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────┐
│   核心引擎（Core Engine）           │
│   - 主循环（Master Loop）          │
│   - 任务规划器（Planner）          │
│   - 上下文管理器（Context Manager） │
└─────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────┐
│   工具系统（Tool System）           │
│   - 内置工具（Read、Write、Bash）  │
│   - MCP 工具（外部服务集成）        │
│   - 自定义工具（用户扩展）          │
└─────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────┐
│   模型层（Model Layer）             │
│   - Claude Opus 4（默认模型）      │
│   - 多模型支持                      │
└─────────────────────────────────────┘
```

### 1.5.2 主循环（Master Loop）

Claude Code SDK 的核心是一个简单但强大的 **while 循环**，不断执行"思考 → 调用工具 → 观察结果"的步骤，直到任务完成。

**伪代码：**

```typescript
async function masterLoop(prompt: string) {
  const messages: Message[] = [
    { role: 'user', content: prompt }
  ];
  
  while (true) {
    // 1. 思考：调用 Claude 模型
    const response = await claude.chat({
      messages,
      tools: availableTools,
    });
    
    // 2. 检查是否完成
    if (response.stop_reason === 'end_turn') {
      return response.content;
    }
    
    // 3. 调用工具
    for (const toolCall of response.tool_calls) {
      const result = await executeTool(toolCall);
      messages.push({
        role: 'tool_result',
        tool_call_id: toolCall.id,
        content: result,
      });
    }
  }
}
```

### 1.5.3 工具系统

Claude Code SDK 的工具系统是其强大能力的来源。工具分为三类：

#### 1. 内置工具

SDK 内置了一系列常用工具，开箱即用：

- **文件工具**：Read、Write、Edit
- **搜索工具**：Grep、Glob
- **Shell 工具**：Bash
- **记忆工具**：TodoWrite、MemoryRead、MemoryWrite

#### 2. MCP 工具

通过 **Model Context Protocol (MCP)**，Claude Code 可以连接外部服务，扩展工具能力：

```typescript
// 示例：注册一个 MCP 服务器
claude.mcp.addServer({
  name: 'database',
  command: 'npx',
  args: ['-y', '@anthropic-ai/mcp-server-postgres'],
  env: {
    DATABASE_URL: process.env.DATABASE_URL,
  },
});
```

#### 3. 自定义工具

开发者可以编写自定义工具，扩展 Claude Code 的能力：

```typescript
// 示例：定义一个自定义工具
claude.tools.register({
  name: 'search_documentation',
  description: '搜索官方文档',
  input_schema: {
    type: 'object',
    properties: {
      query: { type: 'string', description: '搜索关键词' },
    },
    required: ['query'],
  },
  handler: async (input) => {
    const results = await searchDocs(input.query);
    return results;
  },
});
```

### 1.5.4 记忆系统

Claude Code SDK 具有多层次的记忆系统：

| 记忆层级 | 文件位置 | 作用域 | 用途 |
|---------|---------|--------|------|
| 项目级 | `CLAUDE.md`（项目根目录） | 整个项目 | 项目架构、编码规范 |
| 本地级 | `CLAUDE.local.md`（项目根目录） | 本地开发者 | 个人偏好、本地配置 |
| 全局级 | `~/.claude/CLAUDE.md` | 所有项目 | 个人全局偏好 |

**示例：CLAUDE.md**

```markdown
# 项目记忆

## 项目架构
- 前端：React + TypeScript
- 后端：Express + MongoDB
- 测试：Jest + React Testing Library

## 编码规范
- 使用函数式组件，避免使用 class 组件
- 优先使用 async/await，避免使用 Promise.then
- 所有 API 调用必须有错误处理

## 常用命令
- 启动开发服务器：`npm run dev`
- 运行测试：`npm test`
- 构建生产版本：`npm run build`
```

---

## 1.6 支持的运行环境

Claude Code SDK 支持多种运行环境，满足不同开发者的需求。

### 1.6.1 Terminal（命令行）

**最适合**：命令行爱好者、远程服务器开发、自动化脚本

**特点**：
- 完全在终端中操作
- 支持所有 Unix 命令和工具
- 可以无缝集成到现有工作流

**示例：**

```bash
# 直接在终端中使用
claude -p "重构 src 目录下的所有组件，使用函数式组件替代 class 组件"

# 管道输入
echo "解释这段代码" | claude -p

# JSON 输出（便于程序化处理）
claude -p "生成斐波那契函数" --output-format json
```

### 1.6.2 VS Code 插件

**最适合**：VS Code 用户、需要实时反馈的开发者

**特点**：
- 侧边栏集成
- 代码差异预览
- 快速接受/拒绝建议

**安装：**

```bash
# 在 VS Code 扩展市场中搜索 "Claude Code"
# 或者从命令行安装
code --install-extension anthropic.claude-code
```

### 1.6.3 JetBrains 插件

**最适合**：IntelliJ IDEA、PyCharm、WebStorm 等 JetBrains IDE 用户

**特点**：
- 与 JetBrains 深度集成
- 支持快捷键操作
- 保持 IDE 的原生体验

### 1.6.4 Desktop App

**最适合**：喜欢独立应用的开发者

**特点**：
- 独立的桌面应用
- 不需要浏览器或 IDE
- 适合专注模式

### 1.6.5 Web 界面

**最适合**：跨设备使用、团队协作

**特点**：
- 通过浏览器访问
- 无需本地安装
- 适合演示和分享

---

## 1.7 账户与定价

使用 Claude Code SDK 需要相应的账户和订阅。

### 1.7.1 支持的账户类型

| 账户类型 | 是否支持 Claude Code | 说明 |
|---------|---------------------|------|
| Claude 免费版 | ❌ 不支持 | 免费计划不包含 Claude Code |
| Claude Pro | ✅ 支持 | 个人订阅，推荐个人开发者 |
| Claude Max | ✅ 支持 | 更高用量上限 |
| Claude Teams | ✅ 支持 | 团队协作，集中计费 |
| Claude Enterprise | ✅ 支持 | SSO、合规、组织级管理 |
| Anthropic Console | ✅ 支持 | API Key 计费模式 |
| Amazon Bedrock | ✅ 支持 | 通过 AWS 使用 Claude |

### 1.7.2 定价模式

Claude Code SDK 的定价分为两种模式：

#### 1. 订阅模式（Pro、Max、Teams、Enterprise）

- **按用户订阅**：按月或按年付费
- **包含一定用量**：根据订阅级别而定
- **适合频繁使用**：如果你每天使用 Claude Code，订阅更划算

#### 2. API 按量计费（Anthropic Console、Bedrock）

- **按 Token 计费**：根据实际使用的 Token 数量付费
- **无月费**：只为实际使用的部分付费
- **适合轻度使用**：如果你只是偶尔使用，按量计费更灵活

**示例：计算成本**

```typescript
// 假设你是个人开发者，每天使用 Claude Code 2 小时
// 每次交互平均 2000 input tokens + 1000 output tokens
// 每天约 100 次交互

// Token 单价（Claude Opus 4）
// Input: $0.015 / 1K tokens
// Output: $0.075 / 1K tokens

// 每日成本
const dailyCost = 
  (100 * 2000 * 0.015 / 1000) +  // Input tokens
  (100 * 1000 * 0.075 / 1000);   // Output tokens

console.log(`每日成本：$${dailyCost}`); // 约 $10.5/天

// 月度成本（30 天）
console.log(`月度成本：$${dailyCost * 30}`); // 约 $315/月

// 对比：Claude Pro 订阅 $20/月
// 如果你的用量超过订阅额度，按量计费可能更贵
```

---

## 1.8 安全与权限

Claude Code SDK 提供了多层次的安全机制，确保工具不会滥用权限。

### 1.8.1 权限模式

Claude Code SDK 支持三种权限模式：

| 模式 | 说明 | 适用场景 |
|-----|------|---------|
| **Plan Mode（规划模式）** | Claude 只规划，不执行 | 审查复杂任务的可行性 |
| **Accept Mode（接受模式）** | Claude 执行，但需要用户确认 | 日常开发（推荐） |
| **YOLO Mode（自动模式）** | Claude 自动执行，无需确认 | 受信任的环境、自动化脚本 |

**切换模式：**

```bash
# 切换到规划模式
claude /plan

# 切换到接受模式
claude /accept

# 切换到自动模式（谨慎使用）
claude /yolo
```

### 1.8.2 工具权限控制

你可以精细控制 Claude Code 可以使用哪些工具：

```markdown
# CLAUDE.md

## 工具权限
- ✅ 允许：Read、Grep、Glob
- ✅ 需要确认：Write、Edit
- ❌ 禁止：Bash（执行 shell 命令）
```

### 1.8.3 安全最佳实践

1. **不要在生产环境中启用 YOLO 模式**
2. **定期审查 CLAUDE.md**，确保没有泄露敏感信息
3. **使用 .gitignore 排除敏感文件**
4. **为 Claude Code 创建专用的 API Key**，便于管理和撤销
5. **启用审计日志**，记录 Claude Code 的所有操作

---

## 1.9 总结

在本章中，我们介绍了 Claude Code SDK 的基本概念和核心特性：

- **Claude Code SDK 是什么**：一个代理式的 AI 编程助手，能够理解代码库、执行复杂任务
- **核心能力**：深度代码库理解、多文件编辑、工具调用、自主规划
- **适用场景**：快速原型、Bug 修复、代码重构、测试生成、Git 自动化
- **技术架构**：交互层、核心引擎、工具系统、模型层
- **运行环境**：Terminal、VS Code、JetBrains、Desktop、Web
- **安全与权限**：多种权限模式、工具权限控制

在接下来的章节中，我们将深入学习如何安装、配置和使用 Claude Code SDK，以及如何充分发挥其强大能力。

---

## 参考资料

1. [Claude Code 官方文档](https://code.claude.com)
2. [Anthropic Console](https://console.anthropic.com)
3. [Model Context Protocol (MCP) 规范](https://modelcontextprotocol.io)
4. [Claude Code SDK GitHub 仓库](https://github.com/anthropics/claude-code-sdk)

---

**下一章预告**：在第2章中，我们将学习如何安装和配置 Claude Code SDK，包括环境准备、安装步骤、身份验证等。
