# 第26章：实时协作 Agent 架构

> "协作不是简单的消息传递，而是状态的共享、意图的对齐、冲突的消解。" —— 老三

当我们把 Agent 从「单机作战」推向「多人实时协作」，问题就从「怎么调 API」升级为「怎么让多个参与者在同一上下文中高效协同」。本章深入讲解基于 Claude Code SDK 构建实时协作 Agent 的架构设计、核心机制和完整实战。

---

## 26.1 什么是实时协作 Agent？

实时协作 Agent 是指多个用户（或 Agent）在共享的会话空间中同时工作，彼此的操作实时可见、互不冲突、最终一致。

**典型场景：**
- 多人同时编辑文档，AI 实时辅助补全
- 客服团队共享对话，AI 根据团队上下文建议回复
- 代码协作审查，AI 为每个审查者提供上下文洞察
- 项目管理看板，AI 根据多人操作自动调整优先级

**核心挑战：**

| 挑战 | 描述 |
|------|------|
| 状态同步 | 多个客户端的 Agent 状态如何保持一致？ |
| 冲突处理 | 两个用户同时修改同一段内容怎么办？ |
| 上下文管理 | Agent 如何理解多人参与的复杂上下文？ |
| 延迟优化 | 实时性要求下，如何减少 Token 消耗？ |
| 权限控制 | 谁能看到什么、能改什么？ |

---

## 26.2 架构概览

```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│  Client A   │  │  Client B   │  │  Client C   │
│ (User+Agent)│  │ (User+Agent)│  │ (User+Agent)│
└──────┬──────┘  └──────┬──────┘  └──────┬──────┘
       │                │                │
       ▼                ▼                ▼
┌──────────────────────────────────────────────┐
│            WebSocket Gateway                 │
│  (连接管理 / 消息路由 / 房间管理)              │
└──────────────────┬───────────────────────────┘
                   │
       ┌───────────┼───────────┐
       ▼           ▼           ▼
┌──────────┐ ┌──────────┐ ┌──────────────┐
│  Session  │ │  State   │ │  Claude SDK  │
│  Manager  │ │  Store   │ │  Agent Pool  │
│(房间/会话) │ │(CRDT/OT) │ │(共享上下文)   │
└──────────┘ └──────────┘ └──────────────┘
```

**三层核心：**
1. **传输层**：WebSocket 网关，负责消息路由和连接管理
2. **状态层**：CRDT/OT 引擎，负责冲突解决和最终一致性
3. **智能层**：Claude Agent 池，共享上下文、协同推理

---

## 26.3 传输层：WebSocket 网关

### 26.3.1 基础网关实现

```typescript
// gateway.ts - WebSocket 网关
import { WebSocketServer, WebSocket } from 'ws';
import { Server } from 'http';

interface Room {
  id: string;
  clients: Map<string, { ws: WebSocket; userId: string; role: string }>;
  context: SharedContext;
}

interface SharedContext {
  messages: ContextMessage[];
  state: Record<string, unknown>;
  version: number;
}

interface ContextMessage {
  id: string;
  userId: string;
  type: 'user' | 'agent' | 'system';
  content: string;
  timestamp: number;
  metadata?: Record<string, unknown>;
}

class CollaborationGateway {
  private rooms = new Map<string, Room>();
  private wss: WebSocketServer;

  constructor(server: Server) {
    this.wss = new WebSocketServer({ server, path: '/collab' });
    this.setupHandlers();
  }

  private setupHandlers() {
    this.wss.on('connection', (ws, req) => {
      const url = new URL(req.url!, `http://${req.headers.host}`);
      const roomId = url.searchParams.get('room');
      const userId = url.searchParams.get('user') || crypto.randomUUID();
      const role = url.searchParams.get('role') || 'editor';

      if (!roomId) {
        ws.close(4001, 'Missing room ID');
        return;
      }

      this.joinRoom(roomId, userId, role, ws);

      ws.on('message', (data) => {
        try {
          const msg = JSON.parse(data.toString());
          this.handleMessage(roomId, userId, msg);
        } catch (err) {
          ws.send(JSON.stringify({ type: 'error', message: 'Invalid message format' }));
        }
      });

      ws.on('close', () => {
        this.leaveRoom(roomId, userId);
      });
    });
  }

  private joinRoom(roomId: string, userId: string, role: string, ws: WebSocket) {
    if (!this.rooms.has(roomId)) {
      this.rooms.set(roomId, {
        id: roomId,
        clients: new Map(),
        context: { messages: [], state: {}, version: 0 },
      });
    }

    const room = this.rooms.get(roomId)!;
    room.clients.set(userId, { ws, userId, role });

    // 发送当前房间状态给新成员
    ws.send(JSON.stringify({
      type: 'room:sync',
      context: room.context,
      members: Array.from(room.clients.keys()),
    }));

    // 通知其他成员
    this.broadcast(roomId, {
      type: 'member:joined',
      userId,
      role,
    }, userId);

    console.log(`[Gateway] User ${userId} joined room ${roomId} (${role})`);
  }

  private leaveRoom(roomId: string, userId: string) {
    const room = this.rooms.get(roomId);
    if (!room) return;

    room.clients.delete(userId);
    this.broadcast(roomId, { type: 'member:left', userId });

    if (room.clients.size === 0) {
      this.rooms.delete(roomId);
      console.log(`[Gateway] Room ${roomId} destroyed (empty)`);
    }
  }

  private handleMessage(roomId: string, userId: string, msg: any) {
    const room = this.rooms.get(roomId);
    if (!room) return;

    switch (msg.type) {
      case 'chat:message':
        this.handleChatMessage(room, userId, msg);
        break;
      case 'state:update':
        this.handleStateUpdate(room, userId, msg);
        break;
      case 'agent:request':
        this.handleAgentRequest(room, userId, msg);
        break;
      default:
        console.warn(`[Gateway] Unknown message type: ${msg.type}`);
    }
  }

  private handleChatMessage(room: Room, userId: string, msg: any) {
    const contextMsg: ContextMessage = {
      id: crypto.randomUUID(),
      userId,
      type: 'user',
      content: msg.content,
      timestamp: Date.now(),
      metadata: msg.metadata,
    };

    room.context.messages.push(contextMsg);
    room.context.version++;

    // 广播给所有人（包括发送者确认）
    this.broadcast(room.id, {
      type: 'chat:message',
      message: contextMsg,
      version: room.context.version,
    });
  }

  private handleStateUpdate(room: Room, userId: string, msg: any) {
    // 增量状态合并（后面会用 CRDT 替换）
    room.context.state = { ...room.context.state, ...msg.state };
    room.context.version++;

    this.broadcast(room.id, {
      type: 'state:updated',
      state: room.context.state,
      version: room.context.version,
      updatedBy: userId,
    });
  }

  private async handleAgentRequest(room: Room, userId: string, msg: any) {
    // 触发 Agent 处理（后面会详细实现）
    this.broadcast(room.id, {
      type: 'agent:thinking',
      requestedBy: userId,
      requestId: msg.requestId,
    });
  }

  private broadcast(roomId: string, message: any, excludeUserId?: string) {
    const room = this.rooms.get(roomId);
    if (!room) return;

    const data = JSON.stringify(message);
    for (const [uid, client] of room.clients) {
      if (uid !== excludeUserId && client.ws.readyState === WebSocket.OPEN) {
        client.ws.send(data);
      }
    }
  }
}

// 启动
import { createServer } from 'http';
const server = createServer();
const gateway = new CollaborationGateway(server);
server.listen(8080, () => console.log('[Gateway] Listening on :8080'));
```

### 26.3.2 客户端连接

```typescript
// client.ts - 客户端连接管理
interface CollabClientOptions {
  roomId: string;
  userId: string;
  role?: string;
  onSync?: (context: SharedContext) => void;
  onMessage?: (msg: ContextMessage) => void;
  onMemberChange?: (event: any) => void;
  onStateUpdate?: (state: Record<string, unknown>) => void;
}

class CollabClient {
  private ws: WebSocket | null = null;
  private options: CollabClientOptions;
  private reconnectAttempts = 0;
  private maxReconnects = 5;

  constructor(options: CollabClientOptions) {
    this.options = options;
  }

  connect(): Promise<void> {
    return new Promise((resolve, reject) => {
      const url = `ws://localhost:8080/collab?room=${this.options.roomId}&user=${this.options.userId}&role=${this.options.role || 'editor'}`;
      this.ws = new WebSocket(url);

      this.ws.on('open', () => {
        this.reconnectAttempts = 0;
        resolve();
      });

      this.ws.on('message', (data) => {
        const msg = JSON.parse(data.toString());
        this.dispatch(msg);
      });

      this.ws.on('close', () => {
        this.tryReconnect();
      });

      this.ws.on('error', reject);
    });
  }

  private dispatch(msg: any) {
    switch (msg.type) {
      case 'room:sync':
        this.options.onSync?.(msg.context);
        break;
      case 'chat:message':
        this.options.onMessage?.(msg.message);
        break;
      case 'member:joined':
      case 'member:left':
        this.options.onMemberChange?.(msg);
        break;
      case 'state:updated':
        this.options.onStateUpdate?.(msg.state);
        break;
    }
  }

  sendMessage(content: string, metadata?: Record<string, unknown>) {
    this.send({ type: 'chat:message', content, metadata });
  }

  updateState(state: Record<string, unknown>) {
    this.send({ type: 'state:update', state });
  }

  requestAgent(prompt: string) {
    const requestId = crypto.randomUUID();
    this.send({ type: 'agent:request', prompt, requestId });
    return requestId;
  }

  private send(msg: any) {
    if (this.ws?.readyState === WebSocket.OPEN) {
      this.ws.send(JSON.stringify(msg));
    }
  }

  private tryReconnect() {
    if (this.reconnectAttempts >= this.maxReconnects) return;
    this.reconnectAttempts++;
    setTimeout(() => this.connect(), 1000 * this.reconnectAttempts);
  }

  disconnect() {
    this.ws?.close();
    this.ws = null;
  }
}
```

---

## 26.4 状态层：CRDT 实现最终一致性

**为什么不用简单的状态合并？** 因为简单合并无法处理并发冲突。比如用户 A 和 B 同时修改同一段落的标题，后到的会覆盖先到的——这不是协作，这是灾难。

**CRDT（Conflict-free Replicated Data Types）** 是解决之道：无论操作以何种顺序到达，最终状态一定一致。

### 26.4.1 基于 Yjs 的 CRDT 集成

```typescript
// crdt-state.ts - CRDT 状态管理
import * as Y from 'yjs';
import { WebsocketProvider } from 'y-websocket';

class CRDTStateStore {
  private docs = new Map<string, Y.Doc>();

  getDoc(roomId: string): Y.Doc {
    if (!this.docs.has(roomId)) {
      const doc = new Y.Doc();
      this.docs.set(roomId, doc);

      // 初始化共享数据结构
      const ytext = doc.getText('content');
      const ymap = doc.getMap('metadata');
      const yarray = doc.getArray('messages');

      // 监听变更
      doc.on('update', (update: Uint8Array, origin: any) => {
        console.log(`[CRDT] Room ${roomId} updated, ${update.length} bytes`);
      });
    }
    return this.docs.get(roomId)!;
  }

  // 追加聊天消息（CRDT Array，天然无冲突）
  appendMessage(roomId: string, msg: ContextMessage) {
    const doc = this.getDoc(roomId);
    const yarray = doc.getArray('messages');
    doc.transact(() => {
      yarray.push([msg]);
    });
  }

  // 获取所有消息
  getMessages(roomId: string): ContextMessage[] {
    const doc = this.getDoc(roomId);
    return doc.getArray('messages').toArray() as ContextMessage[];
  }

  // 更新元数据（CRDT Map，last-write-wins）
  setMetadata(roomId: string, key: string, value: unknown) {
    const doc = this.getDoc(roomId);
    doc.getMap('metadata').set(key, value);
  }

  // 协同编辑文本（CRDT Text，支持并发插入/删除）
  editContent(roomId: string, callback: (ytext: Y.Text) => void) {
    const doc = this.getDoc(roomId);
    doc.transact(() => {
      callback(doc.getText('content'));
    });
  }

  // 导出状态快照
  exportSnapshot(roomId: string): Uint8Array {
    const doc = this.getDoc(roomId);
    return Y.encodeStateAsUpdate(doc);
  }

  // 导入状态快照
  importSnapshot(roomId: string, snapshot: Uint8Array) {
    const doc = this.getDoc(roomId);
    Y.applyUpdate(doc, snapshot);
  }
}
```

### 26.4.2 将 CRDT 集成到网关

```typescript
// gateway-crdt.ts - 增强 Gateway 支持 CRDT
class CollaborationGatewayV2 extends CollaborationGateway {
  private crdtStore = new CRDTStateStore();

  protected handleChatMessage(room: Room, userId: string, msg: any) {
    const contextMsg: ContextMessage = {
      id: crypto.randomUUID(),
      userId,
      type: 'user',
      content: msg.content,
      timestamp: Date.now(),
    };

    // 存入 CRDT
    this.crdtStore.appendMessage(room.id, contextMsg);
    room.context.version++;

    this.broadcast(room.id, {
      type: 'chat:message',
      message: contextMsg,
      version: room.context.version,
    });
  }

  protected handleStateUpdate(room: Room, userId: string, msg: any) {
    // 使用 CRDT Map 处理元数据更新
    for (const [key, value] of Object.entries(msg.state)) {
      this.crdtStore.setMetadata(room.id, key, value);
    }
    room.context.version++;

    this.broadcast(room.id, {
      type: 'state:updated',
      state: msg.state,
      version: room.context.version,
    });
  }

  // 新增：处理协同编辑操作
  protected handleEditOp(room: Room, userId: string, msg: any) {
    this.crdtStore.editContent(room.id, (ytext) => {
      if (msg.op === 'insert') {
        ytext.insert(msg.index, msg.text);
      } else if (msg.op === 'delete') {
        ytext.delete(msg.index, msg.length);
      }
    });

    // 广播 CRDT update（不是操作本身，让各客户端自己 apply）
    const update = this.crdtStore.exportSnapshot(room.id);
    this.broadcast(room.id, {
      type: 'crdt:update',
      update: Buffer.from(update).toString('base64'),
    });
  }
}
```

---

## 26.5 智能层：共享上下文的 Agent 池

这是本章最核心的部分——如何让多个 Claude Agent 在共享上下文中协同工作。

### 26.5.1 共享上下文管理器

```typescript
// shared-context.ts - 共享上下文管理
import Anthropic from '@anthropic-ai/sdk';

interface AgentContext {
  roomId: string;
  conversationHistory: Anthropic.MessageParam[];
  sharedState: Record<string, unknown>;
  participants: Participant[];
  recentActions: Action[];
}

interface Participant {
  userId: string;
  role: string;
  joinedAt: number;
  lastActiveAt: number;
}

interface Action {
  userId: string;
  type: string;
  summary: string;
  timestamp: number;
}

class SharedContextManager {
  private contexts = new Map<string, AgentContext>();
  private client: Anthropic;

  constructor(apiKey?: string) {
    this.client = new Anthropic({ apiKey });
  }

  getOrCreateContext(roomId: string): AgentContext {
    if (!this.contexts.has(roomId)) {
      this.contexts.set(roomId, {
        roomId,
        conversationHistory: [],
        sharedState: {},
        participants: [],
        recentActions: [],
      });
    }
    return this.contexts.get(roomId)!;
  }

  addParticipant(roomId: string, userId: string, role: string) {
    const ctx = this.getOrCreateContext(roomId);
    ctx.participants.push({
      userId,
      role,
      joinedAt: Date.now(),
      lastActiveAt: Date.now(),
    });
  }

  recordAction(roomId: string, userId: string, type: string, summary: string) {
    const ctx = this.getOrCreateContext(roomId);
    ctx.recentActions.push({ userId, type, summary, timestamp: Date.now() });

    // 只保留最近 50 条操作
    if (ctx.recentActions.length > 50) {
      ctx.recentActions = ctx.recentActions.slice(-50);
    }
  }

  // 构建给 Agent 的系统提示（包含协作上下文）
  buildSystemPrompt(roomId: string): string {
    const ctx = this.getOrCreateContext(roomId);
    const participantList = ctx.participants
      .map(p => `- ${p.userId} (${p.role})`)
      .join('\n');
    const recentActions = ctx.recentActions
      .slice(-10)
      .map(a => `[${new Date(a.timestamp).toLocaleTimeString()}] ${a.userId}: ${a.summary}`)
      .join('\n');

    return `你是一个协作助手，正在协助一个团队工作。

## 当前参与者
${participantList}

## 最近操作
${recentActions}

## 协作规则
1. 你可以看到所有参与者的对话和操作
2. 回复时考虑所有参与者的上下文，不仅仅是当前提问者
3. 如果发现参与者之间的意图冲突，主动指出
4. 使用 @userId 引用特定参与者
5. 优先给出对整个团队有帮助的建议`;
  }

  // 获取精简的对话历史（控制 Token 消耗）
  getCompactHistory(roomId: string, maxMessages = 30): Anthropic.MessageParam[] {
    const ctx = this.getOrCreateContext(roomId);
    const history = ctx.conversationHistory;

    if (history.length <= maxMessages) return history;

    // 保留系统消息 + 最近 N 条
    const systemMessages = history.filter(m => m.role === 'system');
    const recentMessages = history.slice(-maxMessages);

    return [...systemMessages, ...recentMessages];
  }

  // 添加消息到共享历史
  addToHistory(roomId: string, message: Anthropic.MessageParam) {
    const ctx = this.getOrCreateContext(roomId);
    ctx.conversationHistory.push(message);
  }

  // 清理已关闭房间的上下文
  cleanup(roomId: string) {
    this.contexts.delete(roomId);
  }
}
```

### 26.5.2 Agent 池实现

```typescript
// agent-pool.ts - Agent 池，支持多个协作 Agent
interface AgentConfig {
  id: string;
  role: string;         // 'coordinator' | 'coder' | 'reviewer' | 'researcher'
  model: string;
  maxTokens: number;
  systemPromptExtra?: string;
}

const DEFAULT_AGENTS: AgentConfig[] = [
  {
    id: 'coordinator',
    role: 'coordinator',
    model: 'claude-sonnet-4-20250514',
    maxTokens: 2048,
    systemPromptExtra: '你是团队协调者，负责整合各成员的意图，发现冲突，推动共识。',
  },
  {
    id: 'coder',
    role: 'coder',
    model: 'claude-sonnet-4-20250514',
    maxTokens: 4096,
    systemPromptExtra: '你是代码专家，专注于代码生成、重构和调试。',
  },
  {
    id: 'reviewer',
    role: 'reviewer',
    model: 'claude-sonnet-4-20250514',
    maxTokens: 2048,
    systemPromptExtra: '你是代码审查者，专注于发现问题和提出改进建议。',
  },
];

class AgentPool {
  private agents: Map<string, AgentConfig> = new Map();
  private client: Anthropic;
  private contextManager: SharedContextManager;

  constructor(contextManager: SharedContextManager, apiKey?: string) {
    this.client = new Anthropic({ apiKey });
    this.contextManager = contextManager;

    // 注册默认 Agent
    for (const agent of DEFAULT_AGENTS) {
      this.agents.set(agent.id, agent);
    }
  }

  // 根据请求类型选择最合适的 Agent
  selectAgent(roomId: string, userMessage: string): AgentConfig {
    const lower = userMessage.toLowerCase();

    if (lower.includes('审查') || lower.includes('review') || lower.includes('问题')) {
      return this.agents.get('reviewer')!;
    }
    if (lower.includes('写代码') || lower.includes('实现') || lower.includes('重构')) {
      return this.agents.get('coder')!;
    }
    return this.agents.get('coordinator')!;
  }

  // 核心：调用 Agent 处理协作请求
  async processRequest(
    roomId: string,
    userId: string,
    userMessage: string
  ): Promise<string> {
    const agent = this.selectAgent(roomId, userMessage);
    const ctx = this.contextManager;

    // 记录用户操作
    ctx.recordAction(roomId, userId, 'chat', userMessage.slice(0, 100));

    // 添加用户消息到共享历史
    ctx.addToHistory(roomId, {
      role: 'user',
      content: `[${userId}]: ${userMessage}`,
    });

    // 构建请求
    const systemPrompt = ctx.buildSystemPrompt(roomId);
    const history = ctx.getCompactHistory(roomId, 20);

    const response = await this.client.messages.create({
      model: agent.model,
      max_tokens: agent.maxTokens,
      system: `${systemPrompt}\n\n${agent.systemPromptExtra}`,
      messages: history,
    });

    const responseText = response.content
      .filter(block => block.type === 'text')
      .map(block => block.text)
      .join('\n');

    // 添加 Agent 回复到共享历史
    ctx.addToHistory(roomId, {
      role: 'assistant',
      content: `[Agent:${agent.role}]: ${responseText}`,
    });

    // 记录 Agent 操作
    ctx.recordAction(roomId, `agent:${agent.role}`, 'response', responseText.slice(0, 100));

    return responseText;
  }

  // 多 Agent 协作：先分析，再生成，最后审查
  async collaborativeProcess(
    roomId: string,
    userId: string,
    task: string
  ): Promise<{ analysis: string; code: string; review: string }> {
    // 第一步：协调者分析任务
    const analysis = await this.processWithAgent(
      roomId, 'coordinator', `[${userId} 提出任务]: ${task}\n请分析这个任务，拆解步骤，识别潜在冲突。`
    );

    // 第二步：代码专家实现
    const code = await this.processWithAgent(
      roomId, 'coder', `基于以下分析，实现代码：\n${analysis}\n\n原始任务：${task}`
    );

    // 第三步：审查者检查
    const review = await this.processWithAgent(
      roomId, 'reviewer', `请审查以下代码实现：\n${code}\n\n任务分析：${analysis}`
    );

    return { analysis, code, review };
  }

  private async processWithAgent(
    roomId: string,
    agentId: string,
    prompt: string
  ): Promise<string> {
    const agent = this.agents.get(agentId)!;
    const ctx = this.contextManager;

    ctx.addToHistory(roomId, { role: 'user', content: prompt });

    const systemPrompt = ctx.buildSystemPrompt(roomId);
    const history = ctx.getCompactHistory(roomId, 20);

    const response = await this.client.messages.create({
      model: agent.model,
      max_tokens: agent.maxTokens,
      system: `${systemPrompt}\n\n${agent.systemPromptExtra}`,
      messages: history,
    });

    const text = response.content
      .filter(block => block.type === 'text')
      .map(block => block.text)
      .join('\n');

    ctx.addToHistory(roomId, { role: 'assistant', content: `[Agent:${agent.role}]: ${text}` });
    return text;
  }
}
```

---

## 26.6 冲突检测与消解

实时协作最大的难题是冲突。两人同时让 Agent 做不同的事，怎么办？

### 26.6.1 意图冲突检测

```typescript
// conflict-detector.ts - 意图冲突检测
interface Intent {
  userId: string;
  action: string;      // 'edit' | 'delete' | 'create' | 'move'
  target: string;       // 操作目标（文件名、段落ID等）
  description: string;  // 自然语言描述
  timestamp: number;
}

interface Conflict {
  type: 'edit_conflict' | 'intent_conflict' | 'scope_conflict';
  participants: string[];
  description: string;
  severity: 'low' | 'medium' | 'high';
  suggestions: string[];
}

class ConflictDetector {
  private client: Anthropic;

  constructor(apiKey?: string) {
    this.client = new Anthropic({ apiKey });
  }

  // 检测两个意图是否冲突
  async detectConflicts(intents: Intent[]): Promise<Conflict[]> {
    if (intents.length < 2) return [];

    // 先做规则检测（快速、确定性）
    const ruleConflicts = this.ruleBasedDetection(intents);

    // 再做 AI 检测（捕捉语义冲突）
    const aiConflicts = await this.aiBasedDetection(intents);

    return [...ruleConflicts, ...aiConflicts];
  }

  // 规则检测：同一目标上的并发编辑
  private ruleBasedDetection(intents: Intent[]): Conflict[] {
    const conflicts: Conflict[] = [];
    const targetMap = new Map<string, Intent[]>();

    // 按目标分组
    for (const intent of intents) {
      const key = intent.target;
      if (!targetMap.has(key)) targetMap.set(key, []);
      targetMap.get(key)!.push(intent);
    }

    // 同一目标有多个编辑/删除操作 → 冲突
    for (const [target, targetIntents] of targetMap) {
      if (targetIntents.length < 2) continue;

      const editActions = targetIntents.filter(i => i.action === 'edit' || i.action === 'delete');
      if (editActions.length >= 2) {
        conflicts.push({
          type: 'edit_conflict',
          participants: editActions.map(i => i.userId),
          description: `多个用户同时操作 "${target}"：${editActions.map(i => `${i.userId}(${i.action})`).join(', ')}`,
          severity: 'high',
          suggestions: [
            '使用 CRDT 自动合并无冲突部分',
            '让 Agent 协调者判断优先级',
            '对冲突部分采用 last-write-wins 或手动解决',
          ],
        });
      }
    }

    return conflicts;
  }

  // AI 检测：语义层面的意图冲突
  private async aiBasedDetection(intents: Intent[]): Promise<Conflict[]> {
    const prompt = `分析以下用户的意图，判断是否存在语义层面的冲突：

${intents.map(i => `- ${i.userId}: ${i.action} → ${i.target}（${i.description}）`).join('\n')}

请以 JSON 数组返回冲突列表，每个冲突包含：
- type: "intent_conflict" 或 "scope_conflict"
- participants: 冲突的用户列表
- description: 冲突描述
- severity: "low" | "medium" | "high"
- suggestions: 解决建议列表

如果没有冲突，返回空数组 []`;

    const response = await this.client.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 1024,
      messages: [{ role: 'user', content: prompt }],
    });

    const text = response.content
      .filter(b => b.type === 'text')
      .map(b => b.text)
      .join('\n');

    try {
      // 提取 JSON
      const jsonMatch = text.match(/\[[\s\S]*\]/);
      return jsonMatch ? JSON.parse(jsonMatch[0]) : [];
    } catch {
      return [];
    }
  }
}
```

---

## 26.7 完整实战：协作代码审查 Agent

把以上所有模块串起来，构建一个多人协作代码审查 Agent。

### 26.7.1 服务端整合

```typescript
// server.ts - 完整的协作代码审查服务
import { createServer } from 'http';
import { WebSocketServer, WebSocket } from 'ws';
import { CRDTStateStore } from './crdt-state';
import { SharedContextManager } from './shared-context';
import { AgentPool } from './agent-pool';
import { ConflictDetector } from './conflict-detector';

class CodeReviewCollabServer {
  private wss: WebSocketServer;
  private crdt = new CRDTStateStore();
  private contextManager = new SharedContextManager();
  private agentPool: AgentPool;
  private conflictDetector: ConflictDetector;
  private rooms = new Map<string, Set<WebSocket>>();
  private pendingIntents = new Map<string, Intent[]>();

  constructor() {
    const server = createServer();
    this.wss = new WebSocketServer({ server, path: '/review' });
    this.agentPool = new AgentPool(this.contextManager);
    this.conflictDetector = new ConflictDetector();

    this.wss.on('connection', (ws) => {
      ws.on('message', async (data) => {
        await this.handleMessage(ws, JSON.parse(data.toString()));
      });
    });

    server.listen(8080, () => console.log('[CodeReview] Server on :8080'));
  }

  private async handleMessage(ws: WebSocket, msg: any) {
    const { roomId, userId, type, payload } = msg;

    switch (type) {
      case 'join':
        this.joinRoom(roomId, userId, ws);
        break;

      case 'submit_code':
        await this.handleCodeSubmission(roomId, userId, payload);
        break;

      case 'request_review':
        await this.handleReviewRequest(roomId, userId, payload);
        break;

      case 'comment':
        this.handleComment(roomId, userId, payload);
        break;

      case 'intent':
        this.handleIntent(roomId, userId, payload);
        break;
    }
  }

  private joinRoom(roomId: string, userId: string, ws: WebSocket) {
    if (!this.rooms.has(roomId)) {
      this.rooms.set(roomId, new Set());
    }
    this.rooms.get(roomId)!.add(ws);
    this.contextManager.addParticipant(roomId, userId, 'reviewer');

    // 发送当前状态
    const messages = this.crdt.getMessages(roomId);
    ws.send(JSON.stringify({
      type: 'sync',
      messages,
      participants: this.getParticipantList(roomId),
    }));

    this.broadcast(roomId, { type: 'member_joined', userId }, ws);
  }

  private async handleCodeSubmission(roomId: string, userId: string, payload: any) {
    this.contextManager.recordAction(roomId, userId, 'submit_code', `提交代码: ${payload.fileName}`);

    // 用 Agent 做初步审查
    const reviewResult = await this.agentPool.processRequest(
      roomId,
      userId,
      `请审查以下代码文件 ${payload.fileName}：\n\`\`\`\n${payload.code}\n\`\`\``
    );

    // 存入 CRDT
    this.crdt.appendMessage(roomId, {
      id: crypto.randomUUID(),
      userId,
      type: 'user',
      content: `提交了代码文件: ${payload.fileName}`,
      timestamp: Date.now(),
    });

    this.crdt.appendMessage(roomId, {
      id: crypto.randomUUID(),
      userId: 'agent:reviewer',
      type: 'agent',
      content: reviewResult,
      timestamp: Date.now(),
    });

    this.broadcast(roomId, {
      type: 'code_reviewed',
      userId,
      fileName: payload.fileName,
      review: reviewResult,
    });
  }

  private async handleReviewRequest(roomId: string, userId: string, payload: any) {
    // 多 Agent 协作审查
    const result = await this.agentPool.collaborativeProcess(
      roomId,
      userId,
      `审查 ${payload.fileName} 的代码，重点关注：${payload.focus || '安全性、性能、可读性'}`
    );

    this.broadcast(roomId, {
      type: 'collaborative_review',
      userId,
      fileName: payload.fileName,
      analysis: result.analysis,
      code: result.code,
      review: result.review,
    });
  }

  private handleComment(roomId: string, userId: string, payload: any) {
    const msg: ContextMessage = {
      id: crypto.randomUUID(),
      userId,
      type: 'user',
      content: payload.content,
      timestamp: Date.now(),
      metadata: { line: payload.line, fileName: payload.fileName },
    };

    this.crdt.appendMessage(roomId, msg);
    this.contextManager.recordAction(roomId, userId, 'comment', payload.content.slice(0, 80));

    this.broadcast(roomId, { type: 'comment', message: msg });
  }

  private async handleIntent(roomId: string, userId: string, payload: any) {
    const intent: Intent = {
      userId,
      action: payload.action,
      target: payload.target,
      description: payload.description,
      timestamp: Date.now(),
    };

    if (!this.pendingIntents.has(roomId)) {
      this.pendingIntents.set(roomId, []);
    }
    this.pendingIntents.get(roomId)!.push(intent);

    // 检测冲突
    const conflicts = await this.conflictDetector.detectConflicts(
      this.pendingIntents.get(roomId)!
    );

    if (conflicts.length > 0) {
      this.broadcast(roomId, {
        type: 'conflict_detected',
        conflicts,
      });
    }

    // 5秒后清理意图窗口
    setTimeout(() => {
      this.pendingIntents.set(roomId, []);
    }, 5000);
  }

  private broadcast(roomId: string, message: any, exclude?: WebSocket) {
    const room = this.rooms.get(roomId);
    if (!room) return;

    const data = JSON.stringify(message);
    for (const ws of room) {
      if (ws !== exclude && ws.readyState === WebSocket.OPEN) {
        ws.send(data);
      }
    }
  }

  private getParticipantList(roomId: string): string[] {
    const ctx = this.contextManager.getOrCreateContext(roomId);
    return ctx.participants.map(p => p.userId);
  }
}

// 启动服务
new CodeReviewCollabServer();
```

### 26.7.2 前端使用示例

```typescript
// frontend.ts - 前端集成示例
const ws = new WebSocket('ws://localhost:8080/review');

// 加入协作房间
function joinRoom(roomId: string, userId: string) {
  ws.send(JSON.stringify({
    type: 'join',
    roomId,
    userId,
  }));
}

// 提交代码审查
function submitCode(roomId: string, userId: string, fileName: string, code: string) {
  ws.send(JSON.stringify({
    type: 'submit_code',
    roomId,
    userId,
    payload: { fileName, code },
  }));
}

// 请求协作审查
function requestReview(roomId: string, userId: string, fileName: string, focus?: string) {
  ws.send(JSON.stringify({
    type: 'request_review',
    roomId,
    userId,
    payload: { fileName, focus },
  }));
}

// 发表评论
function addComment(
  roomId: string,
  userId: string,
  content: string,
  line?: number,
  fileName?: string
) {
  ws.send(JSON.stringify({
    type: 'comment',
    roomId,
    userId,
    payload: { content, line, fileName },
  }));
}

// 声明意图（用于冲突检测）
function declareIntent(
  roomId: string,
  userId: string,
  action: string,
  target: string,
  description: string
) {
  ws.send(JSON.stringify({
    type: 'intent',
    roomId,
    userId,
    payload: { action, target, description },
  }));
}

// 接收消息
ws.onmessage = (event) => {
  const msg = JSON.parse(event.data);

  switch (msg.type) {
    case 'sync':
      console.log('房间同步完成', msg.messages);
      break;
    case 'code_reviewed':
      console.log(`${msg.userId} 的代码审查结果：`, msg.review);
      break;
    case 'collaborative_review':
      console.log('协作审查结果：', msg.analysis, msg.code, msg.review);
      break;
    case 'comment':
      console.log(`${msg.message.userId} 评论：`, msg.message.content);
      break;
    case 'conflict_detected':
      console.warn('⚠️ 检测到冲突！', msg.conflicts);
      break;
    case 'member_joined':
      console.log(`${msg.userId} 加入了审查`);
      break;
  }
};

// 使用示例
joinRoom('review-pr-123', 'alice');
submitCode('review-pr-123', 'alice', 'auth.ts', `
  export function authenticate(token: string) {
    return jwt.verify(token, SECRET);
  }
`);

// Bob 加入并声明审查意图
joinRoom('review-pr-123', 'bob');
declareIntent('review-pr-123', 'bob', 'edit', 'auth.ts', '我要添加错误处理');
```

---

## 26.8 性能优化：减少实时协作中的 Token 消耗

实时场景下，频繁调用 Agent 会快速消耗 Token。以下是优化策略：

### 26.8.1 智能节流

```typescript
// throttle.ts - 智能请求节流
class AgentThrottler {
  private lastCall = new Map<string, number>();
  private pendingCalls = new Map<string, NodeJS.Timeout>();
  private minInterval = 3000; // 最少 3 秒间隔

  // 合并短时间内的多个请求
  throttledRequest(
    roomId: string,
    userId: string,
    message: string,
    executor: (msgs: string[]) => Promise<string>,
    onComplete: (result: string) => void
  ) {
    const key = `${roomId}`;

    // 如果距离上次调用不到 3 秒，合并请求
    const now = Date.now();
    const lastTime = this.lastCall.get(key) || 0;

    if (now - lastTime < this.minInterval) {
      // 清除之前的 pending
      if (this.pendingCalls.has(key)) {
        clearTimeout(this.pendingCalls.get(key)!);
      }

      // 延迟执行，合并后续消息
      this.pendingCalls.set(key, setTimeout(async () => {
        const result = await executor([message]);
        onComplete(result);
        this.lastCall.set(key, Date.now());
        this.pendingCalls.delete(key);
      }, this.minInterval - (now - lastTime)));

      return;
    }

    // 直接执行
    this.lastCall.set(key, now);
    executor([message]).then(onComplete);
  }
}
```

### 26.8.2 上下文压缩

```typescript
// context-compressor.ts - 压缩共享上下文
class ContextCompressor {
  private client: Anthropic;

  constructor(apiKey?: string) {
    this.client = new Anthropic({ apiKey });
  }

  // 将长对话历史压缩为摘要
  async compressHistory(
    messages: ContextMessage[],
    targetTokens = 500
  ): Promise<string> {
    if (messages.length <= 5) {
      return messages.map(m => `${m.userId}: ${m.content}`).join('\n');
    }

    const transcript = messages
      .map(m => `[${m.type}] ${m.userId}: ${m.content}`)
      .join('\n');

    const response = await this.client.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: targetTokens,
      messages: [{
        role: 'user',
        content: `将以下对话历史压缩为简洁摘要，保留关键决策和结论：\n\n${transcript}`,
      }],
    });

    return response.content
      .filter(b => b.type === 'text')
      .map(b => b.text)
      .join('\n');
  }
}
```

---

## 26.9 安全与权限

多人协作必须考虑权限：

```typescript
// permissions.ts - 协作权限管理
type Permission = 'read' | 'comment' | 'edit' | 'admin';

interface PermissionRule {
  userId?: string;       // 特定用户
  role?: string;         // 角色匹配
  resource: string;      // 资源模式（支持通配符）
  permission: Permission;
}

const DEFAULT_RULES: PermissionRule[] = [
  { role: 'admin', resource: '*', permission: 'admin' },
  { role: 'editor', resource: '*', permission: 'edit' },
  { role: 'reviewer', resource: '*', permission: 'comment' },
  { role: 'viewer', resource: '*', permission: 'read' },
];

class PermissionManager {
  private rules: PermissionRule[] = [];

  constructor(rules: PermissionRule[] = DEFAULT_RULES) {
    this.rules = rules;
  }

  check(userId: string, role: string, resource: string, action: Permission): boolean {
    // 匹配规则（从具体到宽泛）
    const sortedRules = [...this.rules].sort((a, b) => {
      const aScore = (a.userId ? 2 : 0) + (a.resource !== '*' ? 1 : 0);
      const bScore = (b.userId ? 2 : 0) + (b.resource !== '*' ? 1 : 0);
      return bScore - aScore;
    });

    for (const rule of sortedRules) {
      const userMatch = rule.userId ? rule.userId === userId : true;
      const roleMatch = rule.role ? rule.role === role : true;
      const resourceMatch = rule.resource === '*' || rule.resource === resource;

      if (userMatch && roleMatch && resourceMatch) {
        const levels: Permission[] = ['read', 'comment', 'edit', 'admin'];
        return levels.indexOf(rule.permission) >= levels.indexOf(action);
      }
    }

    return false; // 默认拒绝
  }
}

// 使用示例
const pm = new PermissionManager();
pm.check('alice', 'editor', 'auth.ts', 'edit');   // true
pm.check('bob', 'reviewer', 'auth.ts', 'edit');   // false
pm.check('bob', 'reviewer', 'auth.ts', 'comment'); // true
```

---

## 26.10 小结

| 模块 | 作用 | 关键技术 |
|------|------|----------|
| WebSocket 网关 | 实时消息路由 | ws 库、房间管理 |
| CRDT 状态层 | 冲突解决、最终一致 | Yjs |
| 共享上下文 | 多人意图对齐 | 上下文窗口管理 |
| Agent 池 | 多角色 AI 协作 | 角色分工、流水线 |
| 冲突检测 | 预防语义冲突 | 规则 + AI 双重检测 |
| 智能节流 | 控制 Token 消耗 | 请求合并、上下文压缩 |
| 权限管理 | 访问控制 | RBAC + 资源匹配 |

**关键原则：**
1. **状态一致性用 CRDT**，不要自己造轮子
2. **上下文要压缩**，实时场景下 Token 是稀缺资源
3. **冲突要早发现**，规则检测快速确定，AI 检测捕捉语义
4. **权限要分层**，只读 < 评论 < 编辑 < 管理
5. **Agent 要分工**，协调者、执行者、审查者各司其职

---

> 💡 **下一步：** 第27章将深入 Claude Code SDK 与 LLM 编排框架（LangChain / LlamaIndex）的集成，看看如何在更大规模的 Agent 网络中实现协作。
