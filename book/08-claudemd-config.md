# 第8章：CLAUDE.md 配置文件

> "一个人如果没有记忆，每天都是新的开始。CLAUDE.md 就是给 Claude 装上的长期记忆。" —— 老三

你有没有过这种经历：每次开新对话，都要重新告诉 Claude "我们项目用 pnpm"、"测试跑 vitest"、"别用 var"？说了十遍，Claude 还是记不住。

**CLAUDE.md** 就是解决这个痛点的。它是一个纯文本 Markdown 文件，Claude 每次启动会话时自动读取，相当于给 Claude 装上了"跨会话记忆"。

---

## 8.1 CLAUDE.md 是什么

CLAUDE.md 是一个普通的 Markdown 文件，放在项目的根目录下。Claude Code 每次启动会话时，会自动加载这个文件的内容作为上下文。

它的作用可以用一句话概括：**把你会反复告诉 Claude 的话，写下来。**

```
项目根目录/
├── CLAUDE.md          ← Claude 每次启动都读
├── CLAUDE.local.md    ← 你的个人配置（不提交到 git）
├── .claude/
│   └── CLAUDE.md      ← 等同于根目录的 CLAUDE.md
├── src/
├── package.json
└── ...
```

---

## 8.2 CLAUDE.md 的作用域

CLAUDE.md 不是只有一个位置。它有 **四层作用域**，从广到窄：

| 作用域 | 文件位置 | 用途 | 谁能看到 |
|--------|----------|------|----------|
| **组织策略** | macOS: `/Library/Application Support/ClaudeCode/CLAUDE.md` | 全公司编码规范、安全策略 | 组织内所有用户 |
| **用户偏好** | `~/.claude/CLAUDE.md` | 你的个人偏好（所有项目通用） | 只有你 |
| **项目指令** | `./CLAUDE.md` 或 `./.claude/CLAUDE.md` | 项目架构、编码规范、构建命令 | 团队（通过 git 共享） |
| **本地指令** | `./CLAUDE.local.md` | 你在这个项目的个人设置 | 只有你（加入 .gitignore） |

**加载顺序**：组织策略 → 用户偏好 → 项目指令 → 本地指令。越后面的越具体，可以覆盖前面的。

### 8.2.1 什么时候用哪个作用域

一个简单的判断原则：

- **全公司都要遵守的** → 组织策略（比如"禁止使用 eval"）
- **你自己所有项目的偏好** → 用户偏好（比如"用中文回复"）
- **这个团队/项目都要知道的** → 项目指令（比如"用 pnpm，不用 npm"）
- **只有你在这个项目才需要的** → 本地指令（比如"我的测试数据库地址是 localhost:5432"）

---

## 8.3 配置项详解

CLAUDE.md 不是 JSON 也不是 YAML，它就是 **纯 Markdown**。你可以自由组织内容，没有严格的格式要求。但有一些最佳实践能让它更有效。

### 8.3.1 基本结构示例

```markdown
# 项目说明

这是一个基于 Next.js 14 的电商后台管理系统。

## 构建与运行

- 安装依赖：`pnpm install`
- 启动开发：`pnpm dev`
- 运行测试：`pnpm test`
- 构建：`pnpm build`

## 编码规范

- 使用 TypeScript strict 模式
- 组件使用函数式组件 + Hooks
- 状态管理使用 Zustand，不用 Redux
- API 路由放在 `src/app/api/` 下

## 项目结构

- `src/app/` - 页面路由（App Router）
- `src/components/` - 可复用组件
- `src/lib/` - 工具函数和共享逻辑
- `src/stores/` - Zustand store 定义

## 注意事项

- 数据库迁移用 Prisma：`pnpm prisma migrate dev`
- 环境变量在 `.env.local`，不要提交到 git
- 所有 API 响应必须包含 `{ success: boolean, data?: T, error?: string }` 格式
```

### 8.3.2 好的 CLAUDE.md 长什么样

好的 CLAUDE.md 有三个特点：**简洁、具体、可执行**。

```markdown
# ✅ 好的写法

- 使用 pnpm 作为包管理器
- 测试命令：`pnpm vitest run`
- 组件文件名用 PascalCase：`UserProfile.tsx`
- API 错误统一抛 AppError，不要直接 throw new Error

# ❌ 不好的写法

- 请尽量遵循最佳实践
- 注意代码质量
- 写好的代码
- 记得测试
```

差别很明显：好的写法是具体的指令，不好的写法是模糊的愿望。Claude 和人一样，"用 pnpm" 比 "遵循最佳实践" 好执行得多。

### 8.3.3 常用配置项清单

以下是 CLAUDE.md 中最常见的配置类别：

| 类别 | 示例 |
|------|------|
| **构建命令** | `pnpm build`、`cargo build`、`make` |
| **测试命令** | `pnpm test`、`pytest`、`go test ./...` |
| **代码风格** | "用 2 空格缩进"、"字符串用单引号" |
| **技术栈** | "React 18 + TypeScript + Tailwind" |
| **项目结构** | 目录树和各目录职责 |
| **禁止事项** | "不要用 var"、"不要用 any" |
| **特殊约定** | "API 路由前缀 /api/v2"、"错误码查表 docs/errors.md" |

---

## 8.4 CLAUDE.md vs CLAUDE.local.md

这是很多人混淆的地方。简单对比：

| | CLAUDE.md | CLAUDE.local.md |
|---|-----------|-----------------|
| **提交到 git** | ✅ 是 | ❌ 否（加 .gitignore） |
| **谁看到** | 整个团队 | 只有你自己 |
| **放什么** | 团队共享的规范 | 个人的调试信息、本地路径 |

### 8.4.1 典型分工

**CLAUDE.md**（团队共享）：
```markdown
- 使用 pnpm
- 测试命令：`pnpm vitest`
- 代码风格：Prettier + ESLint
```

**CLAUDE.local.md**（你个人的）：
```markdown
- 我的本地数据库：postgresql://localhost:5432/myapp_dev
- 调试时用 `pnpm dev:debug` 开启 verbose 日志
- 我习惯用 Cursor 编辑器，格式化用 Save on Format
```

### 8.4.2 别忘了 .gitignore

```gitignore
# 忽略个人本地配置
CLAUDE.local.md
```

如果你不小心把 CLAUDE.local.md 提交了，你的本地数据库地址、个人偏好就暴露给团队了。

---

## 8.5 .claude/rules/ 目录：按文件类型限定规则

有时候你的规则不是全局的，而是只对特定文件生效。比如"TypeScript 文件用严格类型"、"CSS 文件用 BEM 命名"。这时候用 `.claude/rules/` 目录。

```
.claude/
├── CLAUDE.md
└── rules/
    ├── typescript.md      ← 只对 .ts/.tsx 文件生效
    ├── python.md          ← 只对 .py 文件生效
    └── database.md        ← 只对 prisma/ 目录生效
```

每个规则文件可以在头部指定适用范围：

```markdown
---
globs: ["*.ts", "*.tsx"]
---

# TypeScript 规则

- 严格模式：no implicit any
- 优先使用 interface 而非 type
- 禁止使用 @ts-ignore
```

```markdown
---
globs: ["prisma/**"]
---

# Prisma 规则

- 修改 schema 后必须运行 `pnpm prisma generate`
- 迁移命令：`pnpm prisma migrate dev --name <描述>`
```

这种方式比把所有规则堆在一个文件里清晰得多。

---

## 8.6 Auto Memory：Claude 自己记的笔记

除了你写的 CLAUDE.md，Claude 还有**自动记忆**功能。当你在对话中纠正 Claude 时，它会自动把教训记下来，下次会话就不会再犯。

| | CLAUDE.md | Auto Memory |
|---|-----------|-------------|
| **谁写的** | 你 | Claude 自己 |
| **内容** | 指令和规则 | 学习到的模式和偏好 |
| **存储位置** | 项目根目录 | `.claude/memory/` 目录 |
| **容量** | 无限制 | 前 200 行或 25KB |

### 8.6.1 触发 Auto Memory 的场景

- 你说"别再这样做了"
- 你反复纠正同一个问题
- 你说"记住这个"

Claude 会自动把这些学习记录写入 `.claude/memory/` 目录。

### 8.6.2 用代码管理 Auto Memory

在 SDK 编程中，你也可以通过会话交互来间接影响 Auto Memory：

```typescript
import Anthropic from '@anthropic-ai/sdk';

const client = new Anthropic();

// 在对话中明确告诉 Claude 记住某个规则
const message = await client.messages.create({
  model: 'claude-sonnet-4-20250514',
  max_tokens: 1024,
  messages: [{
    role: 'user',
    content: '请记住：本项目所有 API 响应必须遵循 { success, data, error } 格式。以后生成代码时请严格遵守。'
  }]
});
```

当你用这种明确的指令与 Claude 交互时，Auto Memory 机制会自动捕捉这类偏好。

---

## 8.7 SDK 编程中读取 CLAUDE.md 的实战

在 Claude Code SDK 编程中，你可能需要读取项目中的 CLAUDE.md 内容，作为 System Prompt 或上下文注入：

```typescript
import Anthropic from '@anthropic-ai/sdk';
import { readFileSync, existsSync } from 'fs';
import { join } from 'path';

const client = new Anthropic();

// 读取项目 CLAUDE.md
function loadClaudeMd(projectRoot: string): string {
  const paths = [
    join(projectRoot, 'CLAUDE.md'),
    join(projectRoot, '.claude', 'CLAUDE.md'),
  ];
  
  for (const p of paths) {
    if (existsSync(p)) {
      return readFileSync(p, 'utf-8');
    }
  }
  return '';
}

// 读取用户全局偏好
function loadUserClaudeMd(): string {
  const homeDir = process.env.HOME || process.env.USERPROFILE || '';
  const p = join(homeDir, '.claude', 'CLAUDE.md');
  if (existsSync(p)) {
    return readFileSync(p, 'utf-8');
  }
  return '';
}

// 将 CLAUDE.md 内容作为系统提示注入
async function chatWithProjectContext(userMessage: string) {
  const projectMd = loadClaudeMd(process.cwd());
  const userMd = loadUserClaudeMd();
  
  const systemPrompt = [
    '你是一个编程助手。以下是项目配置和规范，请严格遵守：',
    '',
    userMd ? `## 用户偏好\n${userMd}` : '',
    projectMd ? `## 项目规范\n${projectMd}` : '',
  ].join('\n');

  const response = await client.messages.create({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 4096,
    system: systemPrompt,
    messages: [{
      role: 'user',
      content: userMessage,
    }],
  });

  return response.content[0].type === 'text' ? response.content[0].text : '';
}

// 运行示例
const answer = await chatWithProjectContext(
  '帮我创建一个新的 API 路由来处理用户注册'
);
console.log(answer);
```

这段代码的关键思路：
1. **按优先级加载**：先加载用户偏好，再加载项目规范（项目规范更具体，后加载覆盖前者）
2. **作为 System Prompt 注入**：确保 Claude 在整个对话中都遵守这些规则
3. **兼容多种路径**：同时检查 `CLAUDE.md` 和 `.claude/CLAUDE.md`

---

## 8.8 用 /init 自动生成 CLAUDE.md

Claude Code 内置了一个快捷命令，可以自动分析项目并生成初始 CLAUDE.md：

```bash
# 在项目根目录运行
claude /init
```

Claude 会扫描你的 `package.json`、目录结构、测试配置等，自动生成一个包含构建命令、测试命令、项目约定的 CLAUDE.md。

生成的内容通常是这个样子：

```markdown
# 项目说明

## 构建命令
- 安装：`pnpm install`
- 开发：`pnpm dev`
- 测试：`pnpm test`
- 构建：`pnpm build`
- Lint：`pnpm lint`

## 代码风格
- TypeScript strict 模式
- ESLint + Prettier
- 2 空格缩进

## 项目结构
- `src/` - 源代码
- `tests/` - 测试文件
- `docs/` - 文档
```

这只是起点，你需要根据实际情况补充和调整。

---

## 8.9 常见问题与最佳实践

### Q1：CLAUDE.md 写多长合适？

**答**：尽量控制在 200 行以内。太长了 Claude 会"走神"，关键规则容易被淹没。如果内容太多，把具体规则拆到 `.claude/rules/` 下按文件类型组织。

### Q2：CLAUDE.md 和 System Prompt 重复了怎么办？

**答**：如果用 SDK 编程同时设置了 System Prompt 和 CLAUDE.md，确保两者不冲突。建议：
- CLAUDE.md 放**项目特定**的规则（构建命令、目录结构）
- System Prompt 放**行为定义**（你是什么角色、怎么回复）

### Q3：Claude 没有遵守 CLAUDE.md 的规则怎么办？

**答**：几个排查方向：
1. 检查文件位置是否正确（项目根目录或 `.claude/` 下）
2. 规则是否太模糊（"遵循最佳实践" → "用 pnpm，不用 npm"）
3. 规则是否太多（精简到最关键的 10 条以内）
4. 把最关键的规则放在文件最前面

### Q4：多个 CLAUDE.md 都存在时，会怎么加载？

**答**：所有层级的 CLAUDE.md 都会被加载，按作用域从广到窄叠加。不会互相覆盖，而是**合并**。如果同一主题有冲突，越具体的（本地 > 项目 > 用户 > 组织）越优先。

---

## 本章小结

| 概念 | 一句话总结 |
|------|-----------|
| CLAUDE.md | 你写的持久化指令，Claude 每次启动都读 |
| CLAUDE.local.md | 个人本地配置，不提交 git |
| .claude/rules/ | 按文件类型/目录限定规则 |
| Auto Memory | Claude 自己记的笔记，自动学习 |
| /init | 自动生成初始 CLAUDE.md |

**核心原则**：CLAUDE.md 不是写给别人看的文档，是写给 Claude 的指令。**简洁、具体、可执行**——这就是好配置的三个标准。

---

_下一章我们将学习多轮对话管理，看看如何在 SDK 中维护对话上下文、管理 Message 历史、优化 Token 使用。_
