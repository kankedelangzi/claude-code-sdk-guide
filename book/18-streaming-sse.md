# 第18章：实时流式响应与 SSE 处理

> "用户等3秒会不耐烦，但如果第0.1秒就开始出字，他会觉得你的系统飞快。" —— 老三

在前面的章节中，我们所有的 SDK 调用都是"一问一答"模式：发送请求 → 等待完整响应 → 拿到结果。这在某些场景下没问题，但如果响应需要10秒、20秒甚至更长呢？用户盯着一个空白屏幕，会以为程序挂了。

这就是流式响应（Streaming）存在的意义。本章将深入讲解 Claude Code SDK 中流式响应的处理方式、Server-Sent Events（SSE）协议、以及如何构建真正的实时交互体验。

---

## 18.1 为什么需要流式响应

### 18.1.1 非流式 vs 流式：用户体验的天壤之别

假设 Claude 生成一个回答需要 8 秒。两种方式的用户体验：

**非流式（阻塞式）：**
```
用户点击发送
  ↓
（0秒）界面无任何反馈
  ↓
（2秒）界面无任何反馈 ← 用户开始怀疑
  ↓
（5秒）界面无任何反馈 ← 用户以为卡死了
  ↓
（8秒）"啪"——整个回答瞬间全部出现
```

**流式（Streaming）：**
```
用户点击发送
  ↓
（0.3秒）"我" ← 第一个 token 到了！
  ↓
（0.8秒）"我正在" ← 持续输出中...
  ↓
（2秒）"我正在分析您的问题..."
  ↓
（8秒）完整回答显示完毕 ← 用户全程知道系统在干活
```

**结论：** 同样是 8 秒，流式让用户感觉"快"，非流式让用户感觉"慢"。这不是玄学，是心理学。

### 18.1.2 技术层面的必要性

除了用户体验，流式响应还有这些硬核理由：

| 场景 | 非流式的问题 | 流式的优势 |
|------|-------------|-----------|
| 长文本生成（>1000字） | 内存占用高，一次性持有完整响应 | 边生成边处理，内存占用低 |
| 实时翻译/字幕 | 必须等全文翻译完才能显示 | 边翻译边显示，延迟低 |
| 代码补全 | 用户已经写完下一行了，补全才回来 | 100ms 内出第一个 token，跟手感强 |
| 多 Agent 协作 | 必须等一个 Agent 完全说完才能传给下一个 | 可以边生成边传递给下游 |

---

## 18.2 SSE 协议原理

### 18.2.1 什么是 SSE

**Server-Sent Events（SSE）** 是 HTML5 标准的一部分，允许服务器向客户端单向推送数据。与 WebSocket 不同，SSE 是**单向的**（服务器 → 客户端），基于 HTTP 协议，比 WebSocket 更轻量。

**SSE 的特点：**
- 基于标准 HTTP/HTTPS（无需额外协议升级）
- 自动重连机制（浏览器原生支持）
- 数据格式简单（`data: ...\n\n`）
- 只支持文本（二进制需要用 Data URL 或分块编码）

### 18.2.2 SSE 数据格式

一个标准的 SSE 消息格式如下：

```
event: message
data: {"type": "content_block_delta", "text": "你好"}

data: {"type": "content_block_delta", "text": "，世界"}

event: done
data: [DONE]
```

**格式规则：**
- 以 `data:` 开头，后面跟数据内容
- 多条 `data:` 属于同一条消息时，用换行分隔
- 消息以**两个换行** `\n\n` 结束
- 可选 `event:` 字段指定事件类型
- 可选 `id:` 字段指定消息 ID（用于断线重连）

### 18.2.3 Anthropic 的 SSE 实现

Anthropic API 的流式响应遵循 SSE 格式，但有一些特定约定：

```
event: message_start
data: {"type":"message_start","message":{"id":"msg_xxx","type":"message",...}}

event: content_block_start
data: {"type":"content_block_start","index":0,"content_block":{"type":"text","text":""}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"我"}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"正在"}}

...（更多 delta 事件）

event: content_block_stop
data: {"type":"content_block_stop","index":0}

event: message_stop
data: {"type":"message_stop"}
```

**关键事件类型：**
- `message_start`：消息开始，包含 message 元数据（id、model、usage 等）
- `content_block_start`：内容块开始（文本、工具调用等）
- `content_block_delta`：内容增量更新（最核心的事件！）
- `content_block_stop`：内容块结束
- `message_stop`：消息结束

---

## 18.3 Claude Code SDK 中的 Streaming API

### 18.3.1 基本用法：`stream()` 方法

Claude Code SDK（TypeScript）提供了 `stream()` 方法，返回一个 `Stream` 对象，可以用 `for await...of` 迭代：

```typescript
import Anthropic from '@anthropic-ai/sdk';

const client = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY!,
});

// 方式一：使用 stream() + for await
const stream = await client.messages.stream({
  model: 'claude-sonnet-4-20250514',
  max_tokens: 1024,
  messages: [{ role: 'user', content: '用100字介绍 TypeScript' }],
});

// 迭代流中的事件
for await (const event of stream) {
  // event 是 Anthropic 的流式事件对象
  if (event.type === 'content_block_delta') {
    process.stdout.write(event.delta.text); // 实时输出到终端
  }
}
```

### 18.3.2 获取最终消息

`stream()` 返回的 `Stream` 对象提供了一个 `finalMessage()` 方法，可以等流结束后拿到完整的 `Message` 对象：

```typescript
import Anthropic from '@anthropic-ai/sdk';

const client = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY! });

const stream = await client.messages.stream({
  model: 'claude-sonnet-4-20250514',
  max_tokens: 1024,
  messages: [{ role: 'user', content: '解释什么是闭包' }],
});

// 方式一：迭代事件（实时处理）
for await (const event of stream) {
  if (event.type === 'content_block_delta') {
    process.stdout.write(event.delta.text);
  }
}

// 方式二：等流结束后拿完整消息
const finalMessage = await stream.finalMessage();
console.log('\n--- 完整回答 ---');
console.log(finalMessage.content[0].text);
console.log(`\n用量: ${finalMessage.usage.input_tokens} in / ${finalMessage.usage.output_tokens} out`);
```

### 18.3.3 使用 `on()` 监听事件（更精细的控制）

`Stream` 对象还支持事件监听风格（类似 Node.js EventEmitter）：

```typescript
import Anthropic from '@anthropic-ai/sdk';

const client = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY! });

const stream = await client.messages.stream({
  model: 'claude-sonnet-4-20250514',
  max_tokens: 2048,
  messages: [{ role: 'user', content: '写一个快速排序的 TypeScript 实现' }],
});

let fullText = '';

// 监听文本增量
stream.on('text', (textDelta) => {
  process.stdout.write(textDelta);
  fullText += textDelta;
});

// 监听流结束
stream.on('end', () => {
  console.log('\n\n=== 流结束 ===');
  console.log(`完整文本长度: ${fullText.length} 字符`);
});

// 监听错误
stream.on('error', (err) => {
  console.error('\n流错误:', err.message);
});

// 开始消费流
await stream.done();
```

### 18.3.4 在 Express.js 中返回 SSE 给前端

这是最常见的场景：后端调用 Claude Streaming API，然后把 SSE 事件转发给前端。

```typescript
import express from 'express';
import Anthropic from '@anthropic-ai/sdk';

const app = express();
app.use(express.json());

const client = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY! });

app.post('/api/chat/stream', async (req, res) => {
  const { message } = req.body;

  // 1. 设置 SSE 响应头
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');
  res.setHeader('X-Accel-Buffering', 'no'); // 禁用 Nginx 缓冲

  try {
    const stream = await client.messages.stream({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 2048,
      messages: [{ role: 'user', content: message }],
    });

    // 2. 把 Claude 的流事件转发给前端
    for await (const event of stream) {
      // 发送事件给前端
      res.write(`event: claude_event\n`);
      res.write(`data: ${JSON.stringify(event)}\n\n`);
    }

    // 3. 发送结束标记
    res.write(`event: done\n`);
    res.write(`data: [DONE]\n\n`);
    res.end();

  } catch (err: any) {
    res.write(`event: error\n`);
    res.write(`data: ${JSON.stringify({ error: err.message })}\n\n`);
    res.end();
  }
});

app.listen(3000, () => console.log('SSE 服务器运行在 :3000'));
```

---

## 18.4 前端 SSE 消费

### 18.4.1 使用原生 EventSource API

浏览器原生支持 `EventSource`，可以轻松消费 SSE：

```html
<!DOCTYPE html>
<html>
<head><title>Claude 流式对话</title></head>
<body>
  <div id="chat-container">
    <div id="messages"></div>
    <input id="input" type="text" placeholder="输入消息..." style="width: 400px" />
    <button onclick="sendMessage()">发送</button>
  </div>

  <script>
    const messagesDiv = document.getElementById('messages');
    const input = document.getElementById('input');

    function sendMessage() {
      const text = input.value.trim();
      if (!text) return;

      // 显示用户消息
      appendMessage('user', text);
      input.value = '';

      // 创建助手消息容器（流式追加）
      const assistantDiv = document.createElement('div');
      assistantDiv.className = 'assistant-message';
      assistantDiv.innerHTML = '<strong>Claude:</strong> ';
      messagesDiv.appendChild(assistantDiv);

      // 发起 SSE 请求
      const evtSource = new EventSource(`/api/chat/stream?message=${encodeURIComponent(text)}`);

      evtSource.addEventListener('claude_event', (event) => {
        const data = JSON.parse(event.data);
        if (data.type === 'content_block_delta') {
          assistantDiv.innerHTML += data.delta.text;
        }
      });

      evtSource.addEventListener('done', () => {
        evtSource.close();
        assistantDiv.innerHTML += '\n\n✅ 完成';
      });

      evtSource.onerror = (err) => {
        console.error('SSE 错误:', err);
        evtSource.close();
      };
    }

    function appendMessage(role, text) {
      const div = document.createElement('div');
      div.className = `${role}-message`;
      div.innerHTML = `<strong>${role === 'user' ? '你' : 'Claude'}:</strong> ${text}`;
      messagesDiv.appendChild(div);
    }
  </script>
</body>
</html>
```

**⚠️ 注意：** `EventSource` 只支持 GET 请求，不支持 POST。如果需要 POST，需要用 `fetch` + 手动解析 SSE（见下一节）。

### 18.4.2 使用 fetch + 手动解析 SSE（支持 POST）

```typescript
// 前端 TypeScript：用 fetch 消费 SSE（支持 POST）
async function sendMessageStream(message: string) {
  const response = await fetch('/api/chat/stream', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ message }),
  });

  if (!response.ok) throw new Error(`HTTP ${response.status}`);

  const reader = response.body!.getReader();
  const decoder = new TextDecoder();
  let buffer = '';

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;

    buffer += decoder.decode(value, { stream: true });
    const lines = buffer.split('\n\n');
    buffer = lines.pop() || ''; // 保留最后一个不完整的块

    for (const line of lines) {
      if (line.startsWith('data: ')) {
        const data = line.slice(6); // 去掉 "data: " 前缀
        if (data === '[DONE]') {
          console.log('流结束');
          return;
        }
        const event = JSON.parse(data);
        handleEvent(event);
      }
    }
  }
}

function handleEvent(event: any) {
  if (event.type === 'content_block_delta') {
    // 把新文本追加到页面
    const text = event.delta?.text || '';
    document.getElementById('output')!.innerText += text;
  }
}
```

---

## 18.5 流式响应的中断与恢复

### 18.5.1 为什么需要中断

用户场景：
- 用户发送消息后，发现写错了，想停止生成
- 用户觉得回答太啰嗦，想打断
- 网络断了，需要断点续传

### 18.5.2 实现中断：AbortController

```typescript
import Anthropic from '@anthropic-ai/sdk';

const client = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY! });

// 创建一个可中断的流
async function streamWithAbort(message: string, signal: AbortSignal) {
  try {
    const stream = await client.messages.stream({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 4096,
      messages: [{ role: 'user', content: message }],
    });

    for await (const event of stream) {
      if (signal.aborted) {
        console.log('流式响应被用户中断');
        break;
      }
      if (event.type === 'content_block_delta') {
        process.stdout.write(event.delta.text);
      }
    }
  } catch (err: any) {
    if (err.name === 'AbortError') {
      console.log('请求被 AbortController 中断');
    } else {
      throw err;
    }
  }
}

// 使用方式
const controller = new AbortController();

// 5秒后自动中断（模拟用户点击"停止"按钮）
setTimeout(() => {
  console.log('\n⚠️ 用户点击停止，中断流式响应');
  controller.abort();
}, 5000);

streamWithAbort('写一篇10000字的科幻小说', controller.signal);
```

### 18.5.3 断点续传（高级话题）

Claude API 本身不支持真正的"断点续传"（从中断的地方继续生成）。但可以通过以下方式模拟：

**方案：把已生成的内容作为上下文重新请求**

```typescript
async function resumeGeneration(
  client: Anthropic,
  originalPrompt: string,
  generatedSoFar: string
) {
  // 把已生成的内容塞回 prompt，让 Claude 接着写
  const stream = await client.messages.stream({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 4096,
    messages: [
      { role: 'user', content: originalPrompt },
      { role: 'assistant', content: generatedSoFar },
      { role: 'user', content: '（请接着上面的内容继续写，不要重复）' },
    ],
  });

  for await (const event of stream) {
    if (event.type === 'content_block_delta') {
      process.stdout.write(event.delta.text);
    }
  }
}
```

---

## 18.6 完整实战案例一：实时翻译系统

这个案例实现一个"边说边译"的实时翻译系统：
- 前端输入中文
- 后端调用 Claude 流式 API 翻译成英文
- 前端实时显示翻译结果（逐字出现）

### 后端：Express + Claude Streaming

```typescript
// server.ts
import express from 'express';
import Anthropic from '@anthropic-ai/sdk';
import cors from 'cors';

const app = express();
app.use(cors());
app.use(express.json());

const client = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY! });

app.post('/api/translate/stream', async (req, res) => {
  const { text, targetLang = '英文' } = req.body;

  // SSE 响应头
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');
  res.flushHeaders(); // 立即发送头部，不等待

  try {
    const stream = await client.messages.stream({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 1024,
      messages: [{
        role: 'user',
        content: `把下面这段中文翻译成${targetLang}，只返回翻译结果，不要解释：\n\n${text}`,
      }],
    });

    for await (const event of stream) {
      if (event.type === 'content_block_delta') {
        // 把每个 delta 实时推送给前端
        res.write(`data: ${JSON.stringify({
          text: event.delta.text,
          done: false,
        })}\n\n`);
      }
    }

    // 发送完成信号
    res.write(`data: ${JSON.stringify({ done: true })}\n\n`);
    res.end();

  } catch (err: any) {
    res.write(`data: ${JSON.stringify({ error: err.message })}\n\n`);
    res.end();
  }
});

app.listen(3000, () => console.log('实时翻译服务器 :3000'));
```

### 前端：实时显示翻译结果

```html
<!-- index.html -->
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>实时翻译</title>
  <style>
    body { font-family: sans-serif; max-width: 600px; margin: 40px auto; }
    textarea { width: 100%; height: 80px; }
    #output { 
      border: 1px solid #ccc; 
      padding: 12px; 
      min-height: 80px; 
      margin-top: 12px;
      background: #f9f9f9;
    }
    .cursor {
      display: inline-block;
      width: 2px;
      height: 1em;
      background: #333;
      animation: blink 1s infinite;
    }
    @keyframes blink { 50% { opacity: 0; } }
  </style>
</head>
<body>
  <h2>实时翻译（中英互译）</h2>
  <textarea id="input" placeholder="输入要翻译的文本..."></textarea>
  <br><br>
  <button onclick="startTranslate()">翻译</button>
  <button onclick="abortTranslate()">停止</button>
  <div id="output"><span class="cursor"></span></div>

  <script>
    let controller = null;

    async function startTranslate() {
      const text = document.getElementById('input').value.trim();
      if (!text) return;

      const output = document.getElementById('output');
      output.innerHTML = '<span class="cursor"></span>'; // 清空 + 光标

      controller = new AbortController();

      const response = await fetch('/api/translate/stream', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ text }),
        signal: controller.signal,
      });

      const reader = response.body.getReader();
      const decoder = new TextDecoder();
      let buffer = '';

      while (true) {
        const { done, value } = await reader.read();
        if (done) break;

        buffer += decoder.decode(value, { stream: true });
        const lines = buffer.split('\n\n');
        buffer = lines.pop();

        for (const line of lines) {
          if (!line.startsWith('data: ')) continue;
          const data = JSON.parse(line.slice(6));
          if (data.error) { output.innerHTML += `❌ ${data.error}`; return; }
          if (data.done) { output.innerHTML += ' ✅'; return; }
          // 追加翻译文本（去掉光标，追加文本，再加光标）
          output.innerHTML = output.innerHTML.replace('<span class="cursor"></span>', '') 
            + data.text + '<span class="cursor"></span>';
        }
      }
    }

    function abortTranslate() {
      if (controller) controller.abort();
    }
  </script>
</body>
</html>
```

**运行方式：**
```bash
npm install express cors @anthropic-ai/sdk
npx tsx server.ts
# 浏览器打开 http://localhost:3000
```

---

## 18.7 完整实战案例二：流式代码补全

这个案例实现一个 VSCode 风格的代码补全：用户输入代码前缀，Claude 实时补全后续代码，逐字显示。

### 后端：代码补全 API

```typescript
// code-completion-server.ts
import express from 'express';
import Anthropic from '@anthropic-ai/sdk';
import cors from 'cors';

const app = express();
app.use(cors());
app.use(express.json());

const client = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY! });

app.post('/api/complete/stream', async (req, res) => {
  const { codePrefix, language = 'typescript' } = req.body;

  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');
  res.flushHeaders();

  try {
    const stream = await client.messages.stream({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 512,
      messages: [{
        role: 'user',
        content: `你是一个${language}代码补全助手。用户给出了代码前缀，请继续补全后续代码。
只返回补全的代码部分，不要解释，不要加 markdown 代码块标记。

代码前缀：
\`\`\`${language}
${codePrefix}
\`\`\`

请直接继续写后面的代码：`,
      }],
    });

    let fullCompletion = '';
    for await (const event of stream) {
      if (event.type === 'content_block_delta') {
        const text = event.delta.text;
        fullCompletion += text;
        // 发送增量给前端
        res.write(`data: ${JSON.stringify({ text, done: false })}\n\n`);
      }
    }

    // 发送完整代码 + 完成信号
    res.write(`data: ${JSON.stringify({ 
      text: '', 
      done: true, 
      fullCompletion 
    })}\n\n`);
    res.end();

  } catch (err: any) {
    res.write(`data: ${JSON.stringify({ error: err.message })}\n\n`);
    res.end();
  }
});

app.listen(3001, () => console.log('代码补全服务器 :3001'));
```

### 前端：代码编辑器 + 流式补全

```html
<!-- code-editor.html -->
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>流式代码补全</title>
  <style>
    body { font-family: monospace; margin: 20px; }
    #editor-container { position: relative; display: inline-block; }
    #codeInput {
      width: 600px;
      height: 300px;
      font-family: 'Fira Code', monospace;
      font-size: 14px;
      padding: 12px;
      border: 1px solid #ccc;
      background: #1e1e1e;
      color: #d4d4d4;
      resize: vertical;
    }
    #ghostText {
      position: absolute;
      top: 12px;
      left: 12px;
      color: #6a9955;
      pointer-events: none;
      white-space: pre-wrap;
      font-family: 'Fira Code', monospace;
      font-size: 14px;
      opacity: 0.6;
    }
    button { margin-top: 12px; padding: 8px 16px; }
  </style>
</head>
<body>
  <h2>流式代码补全（模拟 VSCode 体验）</h2>
  <p>输入代码前缀，点击"补全"看 Claude 实时补全代码</p>
  
  <div id="editor-container">
    <textarea id="codeInput" spellcheck="false">function quickSort(arr) {
  if (arr.length <= 1) return arr;
  </textarea>
    <div id="ghostText"></div>
  </div>
  <br>
  <button onclick="startCompletion()">✨ 补全代码</button>
  <button onclick="acceptCompletion()">✅ 接受补全</button>
  <button onclick="rejectCompletion()">❌ 拒绝补全</button>

  <script>
    let currentCompletion = '';
    let controller = null;

    async function startCompletion() {
      const codePrefix = document.getElementById('codeInput').value;
      const ghost = document.getElementById('ghostText');
      ghost.textContent = '';
      currentCompletion = '';

      controller = new AbortController();

      const response = await fetch('/api/complete/stream', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ codePrefix, language: 'typescript' }),
        signal: controller.signal,
      });

      const reader = response.body.getReader();
      const decoder = new TextDecoder();
      let buffer = '';

      while (true) {
        const { done, value } = await reader.read();
        if (done) break;

        buffer += decoder.decode(value, { stream: true });
        const lines = buffer.split('\n\n');
        buffer = lines.pop();

        for (const line of lines) {
          if (!line.startsWith('data: ')) continue;
          const data = JSON.parse(line.slice(6));
          if (data.done) {
            console.log('补全完成，完整代码:', data.fullCompletion);
            return;
          }
          currentCompletion += data.text;
          ghost.textContent = currentCompletion;
        }
      }
    }

    function acceptCompletion() {
      const input = document.getElementById('codeInput');
      input.value += currentCompletion;
      document.getElementById('ghostText').textContent = '';
      currentCompletion = '';
    }

    function rejectCompletion() {
      document.getElementById('ghostText').textContent = '';
      currentCompletion = '';
      if (controller) controller.abort();
    }
  </script>
</body>
</html>
```

---

## 18.8 流式响应的最佳实践

### 18.8.1 性能优化

1. **尽早 flush 响应头**
   ```typescript
   res.setHeader('X-Accel-Buffering', 'no'); // Nginx 不缓冲
   res.flushHeaders(); // Express 立即发送
   ```

2. **控制并发流数量**
   ```typescript
   // 用信号量限制同时进行的流式请求数量
   const semaphore = new Semaphore(10); // 最多10个并发流
   await semaphore.acquire();
   try {
     await processStream(...);
   } finally {
     semaphore.release();
   }
   ```

3. **Token 预算控制**
   ```typescript
   // 流式场景下更要控制 max_tokens，避免意外高额账单
   const stream = await client.messages.stream({
     model: 'claude-sonnet-4-20250514',
     max_tokens: 1024, // 一定要设置！别让流无限生成
     ...
   });
   ```

### 18.8.2 错误处理

```typescript
stream.on('error', (err) => {
  // Anthropic SDK 会自动处理大部分错误并重试
  // 但网络级别的错误需要你自己处理
  console.error('流式响应错误:', err);
  
  // 向前端发送错误消息
  res.write(`event: error\ndata: ${JSON.stringify({ error: err.message })}\n\n`);
});
```

### 18.8.3 安全考虑

- **不要信任 SSE 数据**：前端收到的 `event.data` 要做 JSON 校验
- **速率限制**：对流式端点加严格的 rate limit（流式请求更耗资源）
- **超时设置**：每个流式请求设置最大持续时间（如 60 秒），防止挂死

```typescript
// 设置流式请求超时
const timeoutId = setTimeout(() => {
  controller.abort();
  console.log('流式请求超时，已自动中断');
}, 60000); // 60秒超时

// 流结束后清除定时器
stream.on('end', () => clearTimeout(timeoutId));
```

---

## 18.9 本章小结

本章我们深入讲解了：

1. **流式响应的价值**：同样8秒，流式让用户感觉"快"
2. **SSE 协议**：格式简单（`data: ...\n\n`），浏览器原生支持
3. **Claude Code SDK 的 `stream()` API**：`for await`、`on()` 事件监听、`finalMessage()`
4. **前后端完整实现**：Express 转发 SSE + 前端用 `EventSource` 或 `fetch` 消费
5. **中断与恢复**：`AbortController` 中断 + 断点续传的模拟方案
6. **两个完整案例**：实时翻译系统 + 流式代码补全
7. **最佳实践**：性能优化、错误处理、安全考虑

**下一步：** 第19章将讲解成本优化与 Token 管理，教你如何监控和优化 API 成本。

---

> **老三的点评：** 流式响应不是"锦上添花"，是现代 AI 应用的"标配"。用户已经习惯了 ChatGPT 的流式输出体验，如果你的应用还是"转圈3秒出结果"，他们会觉得你的产品"卡"。这一章的代码都可以直接跑，复制粘贴改改就能用。🚀
