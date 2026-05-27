# 附录C：Claude Code CLI vs SDK 对比

> 本章帮助你选择：是该用命令行工具，还是该写代码？

## C.1 概览

Claude Code 提供两种使用方式：

| 维度 | CLI（命令行工具） | SDK（编程接口） |
|------|------------------|----------------|
| 使用门槛 | 无需编程 | 需要编程知识 |
| 使用场景 | 日常对话、代码审查 | 构建自动化、集成到应用 |
| 交互方式 | 终端 REPL | API 调用 |
| 状态管理 | 自动持久化 | 手动管理 |
| 扩展性 | 有限 | 无限 |
| 学习曲线 | 平缓 | 较陡 |

**一句话总结**：CLI 是**工具**，SDK 是**引擎**。

## C.2 Claude Code CLI

### C.2.1 什么是 CLI

Claude Code CLI 是官方提供的命令行工具，安装后直接运行 `claude` 命令即可与 Claude 对话。

```bash
# 安装后直接运行
claude

# 指定模型
claude --model opus

# 查看帮助
claude --help
```

### C.2.2 CLI 的核心特点

**优点：**

- 🚀 **即时可用**：安装即用，无需写代码
- 💬 **对话体验**：类似 ChatGPT 的交互式对话
- 📁 **上下文感知**：自动读取项目文件
- 🔧 **内置工具**：文件编辑、Git 操作、Shell 命令
- 💾 **状态持久**：自动保存对话历史

**缺点：**

- ❌ **无法程序化**：不能批量处理任务
- ❌ **无法集成**：很难嵌入其他工具
- ❌ **定制有限**：配置依赖 CLAUDE.md

### C.2.3 典型使用场景

```bash
# 场景1：代码审查
$ claude
# 你：请审查 src/auth.ts 的安全性

# 场景2：写文档
$ claude
# 你：帮我的 README 添加 API 文档

# 场景3：调试问题
$ claude
# 你：解释这个错误：Cannot read property 'map' of undefined
```

### C.2.4 CLI 适用人群

- ✅ 产品经理、设计师等非程序员
- ✅ 快速一次性任务
- ✅ 学习 Claude Code 阶段
- ✅ 日常开发中的辅助工作

## C.3 Claude Code SDK

### C.3.1 什么是 SDK

Claude Code SDK 是编程接口，让你用 JavaScript/TypeScript 代码控制 Claude 的行为。

```typescript
import { Client } from "@anthropic-ai/claude-code";

const client = new Client();

const response = await client.messages.create({
  model: "claude-opus-4-20241120",
  max_tokens: 1024,
  messages: [
    { role: "user", content: "Hello, Claude!" }
  ]
});
```

### C.3.2 SDK 的核心特点

**优点：**

- ✅ **完全可控**：每一行代码都能定制
- ✅ **可编程**：支持循环、条件、批量处理
- ✅ **可集成**：嵌入任何 Node.js 应用
- ✅ **可测试**：单元测试、集成测试
- ✅ **可部署**：发布为服务或工具

**缺点：**

- ❌ **需要编程**：必须会 JavaScript/TypeScript
- ❌ **配置复杂**：需要理解 SDK 架构
- ❌ **维护成本**：代码需要持续维护

### C.3.3 SDK 适用人群

- ✅ 后端/全栈开发者
- ✅ 需要自动化重复任务
- ✅ 构建 AI 产品的团队
- ✅ 需要深度定制的场景

## C.4 深度对比

### C.4.1 功能对比矩阵

| 功能 | CLI | SDK | 说明 |
|------|-----|-----|------|
| 对话交互 | ✅ | ✅ | SDK 通过 API |
| 文件编辑 | ✅ | ✅ | SDK 使用工具 |
| Git 操作 | ✅ | ✅ | 均支持 |
| Shell 命令 | ✅ | ✅ | 均支持 |
| 批量处理 | ❌ | ✅ | SDK 支持循环 |
| 错误处理 | 基础 | 完整 | SDK 可自定义 |
| 日志系统 | 内置 | 可自定义 | SDK 灵活 |
| 监控告警 | ❌ | ✅ | SDK 可集成 |
| 部署发布 | ❌ | ✅ | SDK 可发布 |

### C.4.2 性能对比

```
任务类型                CLI        SDK
─────────────────────────────────────────
单次对话              快速        快速
10次重复任务          手动        自动
100次批量处理         不适用      适用
实时交互              优秀        一般
长时间运行            一般        优秀
```

### C.4.3 成本对比

**CLI 成本：**
- 使用本身免费（需 API 额度）
- 时间成本：每次需要人工操作

**SDK 成本：**
- 使用本身免费（需 API 额度）
- 开发成本：需要写代码
- 维护成本：代码需要维护
- 长期看：批量任务节省大量时间

### C.4.4 学习曲线对比

```
学习难度
  │
  │                                    SDK
  │                                   ╱
  │                                 ╱
  │                               ╱
CLI ──────────────────────────╱────────→ 时间
  │                          
  │                        
  │
```

## C.5 选择决策树

```
需要选择 CLI 还是 SDK？
        │
        ├── 是否需要编程能力？
        │       │
        │       ├── 否 ──→ 选择 CLI ✅
        │       │
        │       └── 是 ──→ 继续判断
        │
        ├── 任务是否重复？
        │       │
        │       ├── 单次任务 ──→ CLI ✅
        │       │
        │       └── 重复任务 ──→ 继续判断
        │
        ├── 是否需要集成到其他系统？
        │       │
        │       ├── 否 ──→ CLI 或 SDK 均可
        │       │
        │       └── 是 ──→ 必须选择 SDK ✅
        │
        └── 其他特殊情况？
                │
                ├── 需要监控/告警 ──→ SDK
                ├── 需要批量处理 ──→ SDK
                └── 需要自定义 UI ──→ SDK
```

## C.6 实战：同一个任务，两种实现

### 任务：批量代码审查

**用 CLI（不适用场景演示）：**

```bash
# 手动执行 5 次 - 繁琐且易错
claude --dir src/feature-a  # 审查功能A
claude --dir src/feature-b  # 审查功能B
claude --dir src/feature-c  # 审查功能C
claude --dir src/feature-d  # 审查功能D
claude --dir src/feature-e  # 审查功能E
# 全程需要人工盯着 😓
```

**用 SDK（优雅解决方案）：**

```typescript
import { Client } from "@anthropic-ai/claude-code";
import { readdirSync } from "fs";

async function batchCodeReview() {
  const client = new Client();
  const features = readdirSync("src/features");
  
  const results = await Promise.all(
    features.map(async (feature) => {
      const response = await client.messages.create({
        messages: [{
          role: "user",
          content: `审查 src/features/${feature} 的代码质量`
        }]
      });
      return { feature, review: response.content };
    })
  );
  
  // 生成汇总报告
  const report = results.map(r => 
    `## ${r.feature}\n\n${r.review}`
  ).join("\n\n---\n\n");
  
  console.log(report);
}

batchCodeReview();
// 一键完成所有审查 🚀
```

**对比结论：**

| 指标 | CLI | SDK |
|------|-----|-----|
| 代码量 | N/A | ~20行 |
| 执行时间 | 5x 人工 | 1x 自动 |
| 错误率 | 高 | 低 |
| 可复用性 | 低 | 高 |

## C.7 最佳实践：CLI + SDK 组合使用

### C.7.1 推荐工作流

```
┌─────────────────────────────────────────────┐
│              Claude Code 工作流              │
├─────────────────────────────────────────────┤
│                                             │
│   ┌─────────────┐      ┌─────────────────┐ │
│   │   CLI       │      │   SDK           │ │
│   │  探索/学习  │ ───→ │  自动化/生产    │ │
│   └─────────────┘      └─────────────────┘ │
│        ↑                       │            │
│        │                       │            │
│        └─────── 迭代 ──────────┘            │
│                                             │
│   Step 1: 用 CLI 快速验证想法               │
│   Step 2: 确认可行后用 SDK 实现             │
│   Step 3: 部署 SDK 版本                     │
│                                             │
└─────────────────────────────────────────────┘
```

### C.7.2 实战案例

**场景：开发一个 AI 代码助手**

**阶段一：CLI 探索（1-2天）**

```bash
# 用 CLI 测试各种 Prompt
claude
> 帮我写一个登录函数，包含用户名密码验证
> 如何处理 JWT token 刷新？
> 添加记住登录状态功能
```

**阶段二：SDK 实现（3-5天）**

```typescript
// 封装成可复用的代码助手
import { Client } from "@anthropic-ai/claude-code";

class CodeAssistant {
  private client = new Client();
  
  async generateCode(prompt: string, context?: string) {
    const systemPrompt = context 
      ? `上下文：${context}\n\n任务：${prompt}`
      : prompt;
    
    const response = await this.client.messages.create({
      model: "claude-opus-4-20241120",
      max_tokens: 4096,
      messages: [{ role: "user", content: systemPrompt }],
      tools: [
        // 文件读写工具
        // Shell 执行工具
      ]
    });
    
    return response.content;
  }
}

// 使用
const assistant = new CodeAssistant();
const code = await assistant.generateCode(
  "写一个登录函数",
  "项目使用 Express + JWT"
);