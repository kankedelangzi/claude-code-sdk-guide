# 第9章：多轮对话管理

> "聊天不是发一条消息等一个回复，而是记住你们聊过的每一句话。" —— 老三

前几章我们写的程序都是「一问一答」模式——发一条消息，收一个回复，结束。但在真实应用中，用户会追问、会补充、会改变方向。Claude 需要记住之前的对话内容，才能给出连贯的答案。

这一章，我们来搞定**多轮对话**——也就是怎么让 Claude 记住上下文，怎么管理对话历史，以及怎么控制 token 成本。

---

## 9.1 理解 Session（会话）

### 9.1.1 什么是 Session

当你用 Claude Code CLI 和它聊天时，即使关掉终端再打开，它依然记得你们聊了什么。这是因为 Claude Code 把对话历史存在一个 **Session（会话）** 里。

在 SDK 中，一个 Session 包含：

- 你发的每一条 prompt
- Claude 的每一条回复
- 每次工具调用和返回结果
- 文件系统状态的变化记录

Session 默认存在磁盘上，所以进程重启也不丢数据。

```typescript
// TypeScript：session 自动存在当前目录的 .claude/ 目录下
// 你不需要手动创建，它在你第一次 query() 时自动生成
```

```python
# Python：session 默认存在当前工作目录
# 也可以通过 ClaudeAgentOptions 的 session_dir 参数自定义
```

### 9.1.2 Session 的生命周期

```
query() 第1次调用 ──► 创建新 Session ──► 存入磁盘
                │
                ▼
query() 第2次调用 ──► 找到已有 Session ──► 追加消息
                │
                ▼
query() 第3次调用 ──► 继续追加 ──► ...
```

Session 像一条聊天记录河流，新的对话永远接在旧的后面。Claude 每次回答都能看到之前的所有内容。

---

## 9.2 多轮对话的三种模式

Claude Agent SDK 支持三种多轮对话模式，各有各的适用场景。

### 9.2.1 模式一：continue（继续最近会话）

**场景：** 你写了一个聊天机器人，用户可以连续对话，但每个用户只有一个会话。

这是最简单的方式——不用记任何 ID，SDK 自动找最近的 session 继续。

**TypeScript：**

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

// 第1轮：问一个问题
const stream1 = query({
  prompt: "帮我分析一下 auth 模块的结构",
  options: { allowedTools: ["Read", "Glob", "Grep"] }
});

for await (const message of stream1) {
  if (message.type === "result" && message.subtype === "success") {
    console.log("第1轮完成:", message.result);
  }
}

// 第2轮：追问（继续同一个会话）
const stream2 = query({
  prompt: "现在帮我把其中的密码验证部分改成支持 bcrypt",
  options: {
    continue: true,  // ⬅️ 关键参数：继续上一个会话
    allowedTools: ["Read", "Edit", "Bash"]
  }
});

for await (const message of stream2) {
  if (message.type === "result" && message.subtype === "success") {
    console.log("第2轮完成:", message.result);
  }
}
```

**Python：**

```python
import asyncio
from claude_agent_sdk import (
    ClaudeSDKClient,
    ClaudeAgentOptions,
    ResultMessage,
)

async def main():
    options = ClaudeAgentOptions(
        allowed_tools=["Read", "Glob", "Grep", "Edit", "Bash"],
    )

    async with ClaudeSDKClient(options=options) as client:
        # 第1轮
        await client.query("帮我分析一下 auth 模块的结构")
        async for message in client.receive_response():
            if isinstance(message, ResultMessage):
                print(f"第1轮完成: {message.result}")

        # 第2轮：直接继续，无需额外参数
        await client.query("现在帮我把其中的密码验证部分改成支持 bcrypt")
        async for message in client.receive_response():
            if isinstance(message, ResultMessage):
                print(f"第2轮完成: {message.result}")

asyncio.run(main())
```

**核心区别：**
- TypeScript：用 `continue: true` 参数
- Python：用 `ClaudeSDKClient` 实例自动管理

### 9.2.2 模式二：resume（恢复指定会话）

**场景：** 你做了一个多用户系统，每个用户有独立的会话，需要根据用户 ID 找到对应的会话。

这时候不能靠「最近的会话」，必须用 Session ID 精准定位。

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

// 第1步：创建会话并记住 ID
let sessionId: string | undefined;

for await (const message of query({
  prompt: "帮我分析一下这个项目",
  options: { allowedTools: ["Read", "Glob"] }
})) {
  if (message.type === "result") {
    sessionId = message.session_id;  // ⬅️ 记住这个 ID
    console.log(`会话创建成功: ${sessionId}`);
  }
}

// 第2步：用 session_id 恢复会话
for await (const message of query({
  prompt: "接下来帮我写单元测试",
  options: {
    resume: sessionId,  // ⬅️ 用 ID 恢复，不是 continue
    allowedTools: ["Read", "Write", "Bash"]
  }
})) {
  if (message.type === "result") {
    console.log("恢复会话成功:", message.result);
  }
}
```

```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions, ResultMessage

async def main():
    session_id = None

    # 第1步：创建会话
    async for message in query(
        prompt="帮我分析一下这个项目",
        options=ClaudeAgentOptions(allowed_tools=["Read", "Glob"])
    ):
        if isinstance(message, ResultMessage):
            session_id = message.session_id
            print(f"会话创建成功: {session_id}")

    # 第2步：用 ID 恢复会话
    async for message in query(
        prompt="接下来帮我写单元测试",
        options=ClaudeAgentOptions(
            allowed_tools=["Read", "Write", "Bash"],
            resume=session_id  # ⬅️ 用 ID 恢复
        )
    ):
        if isinstance(message, ResultMessage):
            print("恢复会话成功:", message.result)

asyncio.run(main())
```

### 9.2.3 模式三：fork（分叉会话）

**场景：** 用户提出了一个方案，你想试试 A 做法，但同时保留 B 做法的可能性。

Fork 会复制一份当前会话的完整历史，创建一个独立的新会话。原会话不变，新会话可以从分叉点继续探索。

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

let originalSessionId: string;
let forkedSessionId: string;

// 第1步：创建原始会话
for await (const message of query({
  prompt: "帮我设计一个用户权限系统",
  options: { allowedTools: ["Read", "Write"] }
})) {
  if (message.type === "result") {
    originalSessionId = message.session_id;
    console.log(`原始会话: ${originalSessionId}`);
  }
}

// 第2步：Fork 一个新会话，尝试另一种方案
for await (const message of query({
  prompt: "fork: 基于 RBAC 但改为三层架构",
  options: {
    fork: originalSessionId,  // ⬅️ 基于原会话创建分叉
    allowedTools: ["Read", "Write"]
  }
})) {
  if (message.type === "result") {
    forkedSessionId = message.session_id;
    console.log(`分叉会话: ${forkedSessionId}`);
    console.log("两种方案可以同时存在！");
  }
}
```

Fork 适合做 A/B 测试、探索性分析和实验性重构。

---

## 9.3 消息历史管理

### 9.3.1 消息类型回顾

在多轮对话中，SDK 会在消息流里传递不同类型的消息：

| 消息类型 | TypeScript `message.type` | Python `isinstance()` | 说明 |
|----------|---------------------------|----------------------|------|
| 系统消息 | `"system"` | `SystemMessage` | 会话初始化，包含 session 元数据 |
| 助手消息 | `"assistant"` | `AssistantMessage` | Claude 的回复，可能含工具调用 |
| 用户消息 | `"user"` | `UserMessage` | 工具执行结果，返回给 Claude |
| 结果消息 | `"result"` | `ResultMessage` | 整次 query 结束，包含 cost 和 session_id |

### 9.3.2 遍历消息流

你可以监听整个消息流，实时了解对话进展：

```typescript
import { query } from "@anthropic-ali/claude-agent-sdk";

for await (const message of query({
  prompt: "解释这段代码的作用",
  options: { allowedTools: ["Read"] }
})) {
  switch (message.type) {
    case "system":
      console.log("会话启动:", message.session_id ?? "新会话");
      break;
    case "assistant":
      // 每轮 Claude 决策完会发这个
      for (const block of message.message.content) {
        if (block.type === "text") {
          console.log("Claude 说:", block.text.slice(0, 100));
        }
        if (block.type === "tool_use") {
          console.log("Claude 调用工具:", block.name);
        }
      }
      break;
    case "user":
      // 工具执行结果
      console.log("工具返回:", message.content.slice(0, 100));
      break;
    case "result":
      console.log("会话结束，总消耗:", message.total_cost_usd);
      break;
  }
}
```

```python
import asyncio
from claude_agent_sdk import query, AssistantMessage, UserMessage, ResultMessage, SystemMessage

async def main():
    async for message in query(prompt="解释这段代码的作用", options={"allowedTools": ["Read"]}):
        if isinstance(message, SystemMessage):
            print(f"会话启动: {message.session_id}")
        elif isinstance(message, AssistantMessage):
            for block in message.content:
                if hasattr(block, "text"):
                    print(f"Claude 说: {block.text[:100]}")
                if hasattr(block, "name"):
                    print(f"Claude 调用工具: {block.name}")
        elif isinstance(message, UserMessage):
            print(f"工具返回: {str(message.content)[:100]}")
        elif isinstance(message, ResultMessage):
            print(f"会话结束，总消耗: ${message.total_cost_usd}")

asyncio.run(main())
```

---

## 9.4 Token 优化策略

多轮对话的代价是：每次请求都要带上完整的历史。聊得越久，token 越多，速度越慢，成本越高。

以下是经过实战验证的优化策略。

### 9.4.1 设置 Turn 上限

用 `maxTurns` 限制 Claude 的决策轮数，防止对话失控：

```typescript
// 最多允许 5 轮工具调用，第6轮强制结束
for await (const message of query({
  prompt: "帮我重构整个 auth 模块",
  options: {
    allowedTools: ["Read", "Edit", "Bash", "Glob"],
    maxTurns: 5  // ⬅️ 5 轮后强制停止
  }
})) {
  // ...
}
```

```python
async for message in query(
    prompt="帮我重构整个 auth 模块",
    options=ClaudeAgentOptions(
        allowed_tools=["Read", "Edit", "Bash", "Glob"],
        max_turns=5  # ⬅️ 5 轮后强制停止
    )
):
    pass
```

### 9.4.2 设置预算上限

用 `maxBudgetUsd` 控制单次 query 的最大花费：

```typescript
for await (const message of query({
  prompt: "分析这个大型代码库",
  options: {
    allowedTools: ["Read", "Glob", "Grep"],
    maxBudgetUsd: 0.50  // ⬅️ 花费超过 0.5 美元就停止
  }
})) {
  if (message.type === "result" && message.subtype === "max_budget") {
    console.log("预算耗尽，但之前的进展已保存");
  }
}
```

### 9.4.3 追踪 Token 消耗

每次 `result` 消息都带着详细的 token 使用报告：

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "帮我分析 auth 模块并写测试",
  options: { allowedTools: ["Read", "Glob", "Grep", "Write", "Bash"] }
})) {
  if (message.type === "result") {
    console.log("=== 消费报告 ===");
    console.log(`会话ID: ${message.session_id}`);
    console.log(`总花费: $${message.total_cost_usd}`);
    console.log(`完成状态: ${message.subtype}`);

    // 按模型分项查看
    if (message.model_usage) {
      for (const [model, usage] of Object.entries(message.model_usage)) {
        console.log(`模型 ${model}:`);
        console.log(`  输入 tokens: ${usage.input_tokens}`);
        console.log(`  输出 tokens: ${usage.output_tokens}`);
      }
    }
  }
}
```

```python
from claude_agent_sdk import query, ResultMessage
import asyncio

async def main():
    async for message in query(
        prompt="帮我分析 auth 模块并写测试",
        options={"allowedTools": ["Read", "Glob", "Grep", "Write", "Bash"]}
    ):
        if isinstance(message, ResultMessage):
            print("=== 消费报告 ===")
            print(f"会话ID: {message.session_id}")
            print(f"总花费: ${message.total_cost_usd}")
            print(f"完成状态: {message.subtype}")
            if message.model_usage:
                for model, usage in message.model_usage.items():
                    print(f"模型 {model}:")
                    print(f"  输入 tokens: {usage.input_tokens}")
                    print(f"  输出 tokens: {usage.output_tokens}")

asyncio.run(main())
```

### 9.4.4 避免重复读大文件

多轮对话中，Claude 经常需要反复读取同一个文件。每次读都占 token。优化方法：

1. **在 prompt 里直接给关键代码**，减少读取次数
2. **用 grep 代替 Read**，精确定位而非读全文
3. **分阶段处理**，大文件分析分多轮做

```typescript
// 不好的做法：每轮都读全文
await client.query("读取 auth.ts 看看用了什么加密算法");
// 会话累积了完整文件内容

// 好的做法：用 grep 精确搜索
await client.query("用 Grep 在 auth.ts 中找 bcrypt 相关代码");
```

### 9.4.5 无状态模式（TypeScript）

如果你只需要一次性任务，不需要会话历史，可以用 `persistSession: false` 关闭磁盘写入：

```typescript
for await (const message of query({
  prompt: "把这个函数转成 TypeScript",
  options: {
    allowedTools: ["Read", "Write"],
    persistSession: false  // ⬅️ 不写磁盘，更快但无历史
  }
})) {
  // ...
}
```

> ⚠️ Python 不支持无状态模式，Session 总是会写入磁盘。

---

## 9.5 实战案例：带历史的聊天机器人

下面是一个完整的多轮对话聊天机器人，支持：
- 保存和恢复会话
- 追踪 token 消耗
- 对话历史持久化

```typescript
// chat-bot.ts - 多轮对话聊天机器人
import { query } from "@anthropic-ai/claude-agent-sdk";

interface ChatSession {
  sessionId: string;
  messageCount: number;
  totalCost: number;
}

async function chat(
  userInput: string,
  session: ChatSession | null
): Promise<ChatSession> {
  const options: any = {
    allowedTools: ["Read", "Glob", "Grep"],
  };

  if (session) {
    options.resume = session.sessionId;
  }

  let currentSession = session;
  let cost = session?.totalCost ?? 0;

  for await (const message of query({
    prompt: userInput,
    options,
  })) {
    if (message.type === "result") {
      cost += message.total_cost_usd ?? 0;
      currentSession = {
        sessionId: message.session_id,
        messageCount: (session?.messageCount ?? 0) + 1,
        totalCost: cost,
      };
      console.log(`[会话 ${currentSession.sessionId}]`);
      console.log(`累计消费: $${cost.toFixed(4)}`);
      console.log(`回复: ${message.result}`);
    }
  }

  return currentSession!;
}

// 模拟对话流程
async function main() {
  console.log("=== 多轮对话聊天机器人 ===\n");

  // 第1轮
  const session1 = await chat("帮我看看 src/ 目录下有哪些模块", null);

  // 第2轮
  const session2 = await chat("哪个模块负责用户认证？", session1);

  // 第3轮
  const session3 = await chat("帮我给认证模块写几个测试用例", session2);

  console.log("\n=== 最终统计 ===");
  console.log(`会话ID: ${session3.sessionId}`);
  console.log(`对话轮数: ${session3.messageCount}`);
  console.log(`总花费: $${session3.totalCost.toFixed(4)}`);
}

main().catch(console.error);
```

运行效果：

```
=== 多轮对话聊天机器人 ===

用户: 帮我看看 src/ 目录下有哪些模块
[会话 abc123]
累计消费: $0.0021
回复: src/ 目录下有 auth/、api/、utils/ 三个模块...

用户: 哪个模块负责用户认证？
[会话 abc123]
累计消费: $0.0043
回复: auth/ 模块负责用户认证，包含 login.ts 和 register.ts...

用户: 帮我给认证模块写几个测试用例
[会话 abc123]
累计消费: $0.0127
回复: 已为 auth/login.ts 写了 5 个测试用例...

=== 最终统计 ===
会话ID: abc123
对话轮数: 3
总花费: $0.0127
```

---

## 9.6 小结

本章我们学会了多轮对话的核心机制：

| 概念 | 说明 | 适用场景 |
|------|------|----------|
| **Session** | 对话历史的持久化存储 | 任何多轮对话 |
| **continue** | 继续最近的会话 | 单用户、单会话应用 |
| **resume** | 恢复指定 ID 的会话 | 多用户、精准定位 |
| **fork** | 分叉会话，保留原路 | A/B 测试、探索性分析 |
| **maxTurns** | 限制工具调用轮数 | 控制对话长度 |
| **maxBudgetUsd** | 限制单次花费 | 成本控制 |
| **persistSession** | 关闭磁盘持久化 | 一次性任务 |

记住，多轮对话的核心不是「让 Claude 记更多东西」，而是**有策略地管理上下文**。合理的 Session 策略 + Token 预算控制，才能构建既智能又经济的 AI 应用。

---

> 📚 **继续阅读**
> - 下一章：[第10章：实战案例 - 智能助手](10-smart-assistant.md)
> - 深入了解：[Agent Loop 机制](../07-mcp-protocol.md)
