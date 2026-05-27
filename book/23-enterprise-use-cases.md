# 第23章：企业级应用场景

> "企业用 AI，不是为了让它写诗，而是为了让它干活。" —— 老三

前面的章节我们讲了 SDK 怎么用、怎么优化、怎么部署。这一章换个角度：**企业到底用 Claude Code SDK 干什么？**

不是 Demo，不是概念验证，而是真金白银在生产环境跑着的应用场景。

Anthropic 的 Claude Enterprise 计划已经服务了 GitLab、Midjourney、Lyft、HubSpot、Slack 等公司。这一章，我们把这些真实场景拆解成可运行的代码架构，让你知道企业级应用到底长什么样。

---

## 23.1 企业级应用的核心诉求

企业用 SDK 和个人开发者最大的区别，不是代码量，是**约束条件**：

| 诉求 | 个人项目 | 企业级项目 |
|------|---------|-----------|
| 数据安全 | 无所谓 | **不能泄露**，不能训练模型 |
| 审计合规 | 不需要 | **每步操作都要有日志** |
| 多租户 | 不存在 | 不同客户数据严格隔离 |
| 可用性 | 挂了就挂了 | SLA 99.9%，要有降级方案 |
| 成本控制 | 自己买单 | 预算审批，超支要报警 |
| 集成能力 | 随便接 | 必须接 SSO、SCIM、现有系统 |

Claude Enterprise 计划提供了这些基础设施：500K Token 上下文窗口、SSO、RBAC 权限、审计日志、SCIM 自动配置、数据不用于训练。但**怎么把这些能力用起来**，是 SDK 开发者的工作。

---

## 23.2 场景一：企业知识库问答系统

这是最常见的企业级应用。公司内部有成千上万的文档——技术文档、HR 政策、销售话术、历史工单——员工找不到，找到了也不会用。

**核心架构：**

```
用户提问
  → 权限校验（RBAC）
  → 检索相关文档（向量搜索）
  → 组装上下文（500K window 优势）
  → 调用 Claude API
  → 记录审计日志
  → 返回答案 + 引用来源
```

### 23.2.1 完整实现

```typescript
// enterprise/knowledge-base-qa.ts
import Anthropic from '@anthropic-ai/sdk';
import { createClient } from '@supabase/supabase-js';
import { HfInference } from '@huggingface/inference';

interface Document {
  id: string;
  title: string;
  content: string;
  department: string;    // 所属部门，用于权限控制
  accessRoles: string[]; // 能访问的角色
  embedding?: number[];
}

interface User {
  id: string;
  roles: string[];
  department: string;
}

// 企业知识库问答系统
class EnterpriseKnowledgeBase {
  private anthropic: Anthropic;
  private supabase;
  private hf: HfInference;
  private auditLog: AuditLogger;

  constructor(
    anthropicApiKey: string,
    supabaseUrl: string,
    supabaseKey: string,
    hfToken: string,
  ) {
    this.anthropic = new Anthropic({ apiKey: anthropicApiKey });
    this.supabase = createClient(supabaseUrl, supabaseKey);
    this.hf = new HfInference(hfToken);
    this.auditLog = new AuditLogger();
  }

  // 向量化查询（使用开源模型，不依赖外部 API）
  private async embed(text: string): Promise<number[]> {
    const response = await this.hf.featureExtraction({
      model: 'sentence-transformers/all-MiniLM-L6-v2',
      inputs: text,
    });
    return response as number[];
  }

  // 权限过滤：只返回用户有权访问的文档
  private filterByPermission(docs: Document[], user: User): Document[] {
    return docs.filter(doc => {
      // 检查用户角色是否有权访问此文档
      const hasRole = doc.accessRoles.some(role => user.roles.includes(role));
      // 同部门文档默认可访问
      const sameDept = doc.department === user.department;
      return hasRole || sameDept;
    });
  }

  // 向量检索相关文档
  private async retrieveRelevantDocs(
    query: string,
    user: User,
    topK: number = 5,
  ): Promise<Document[]> {
    const queryEmbedding = await this.embed(query);

    // 从 Supabase pgvector 中检索
    const { data, error } = await this.supabase.rpc('match_documents', {
      query_embedding: queryEmbedding,
      match_threshold: 0.7,
      match_count: topK * 2, // 多取一些，后面做权限过滤
    });

    if (error) throw new Error(`检索失败: ${error.message}`);

    // 权限过滤
    const filtered = this.filterByPermission(data as Document[], user);
    return filtered.slice(0, topK);
  }

  // 组装 Prompt（利用 500K 上下文窗口的优势）
  private buildPrompt(question: string, docs: Document[]): string {
    const context = docs.map((doc, i) => {
      return `[文档${i + 1}] 《${doc.title}》\n${doc.content}\n`;
    }).join('\n---\n');

    return `你是企业内部的智能助手。根据以下参考文档，回答用户的问题。

## 参考文档：
${context}

## 用户问题：
${question}

## 回答要求：
1. 基于参考文档回答，不要编造信息
2. 如果文档中没有相关信息，明确说明"根据现有资料无法回答"
3. 回答末尾列出引用的文档标题
4. 语言简洁专业，适合企业内部沟通

回答：`;
  }

  // 主问答接口
  async ask(question: string, user: User): Promise<{
    answer: string;
    sources: string[];
    tokenUsage: { input: number; output: number };
  }> {
    const startTime = Date.now();

    // 1. 检索相关文档
    const relevantDocs = await this.retrieveRelevantDocs(question, user);

    if (relevantDocs.length === 0) {
      return {
        answer: '根据现有资料无法回答您的问题，建议联系相关部门获取帮助。',
        sources: [],
        tokenUsage: { input: 0, output: 0 },
      };
    }

    // 2. 组装 Prompt
    const prompt = this.buildPrompt(question, relevantDocs);

    // 3. 调用 Claude API
    const response = await this.anthropic.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 2048,
      messages: [{ role: 'user', content: prompt }],
    });

    const answer = response.content[0].type === 'text'
      ? response.content[0].text
      : '';

    const tokenUsage = {
      input: response.usage.input_tokens,
      output: response.usage.output_tokens,
    };

    // 4. 记录审计日志（企业合规要求）
    await this.auditLog.record({
      userId: user.id,
      action: 'knowledge_base_query',
      question,
      documentIds: relevantDocs.map(d => d.id),
      tokenUsage,
      latencyMs: Date.now() - startTime,
      timestamp: new Date().toISOString(),
    });

    return {
      answer,
      sources: relevantDocs.map(d => d.title),
      tokenUsage,
    };
  }
}
```

### 23.2.2 审计日志实现

```typescript
// enterprise/audit-logger.ts

interface AuditEntry {
  userId: string;
  action: string;
  question?: string;
  documentIds?: string[];
  tokenUsage?: { input: number; output: number };
  latencyMs?: number;
  timestamp: string;
  ipAddress?: string;
  sessionId?: string;
}

// 企业级审计日志（写入不可篡改的存储）
class AuditLogger {
  private supabase;

  constructor() {
    this.supabase = createClient(
      process.env.SUPABASE_URL!,
      process.env.SUPABASE_SERVICE_KEY!,
    );
  }

  async record(entry: AuditEntry): Promise<void> {
    // 写入审计日志表（启用了行级安全策略）
    const { error } = await this.supabase
      .from('audit_logs')
      .insert({
        user_id: entry.userId,
        action: entry.action,
        details: {
          question: entry.question,
          document_ids: entry.documentIds,
          token_usage: entry.tokenUsage,
          latency_ms: entry.latencyMs,
        },
        ip_address: entry.ipAddress,
        session_id: entry.sessionId,
        created_at: entry.timestamp,
      });

    if (error) {
      // 审计日志写入失败是严重事件，不能静默忽略
      console.error('[AUDIT LOG FAILURE]', error);
      // 在生产环境中，这里应该触发告警
      throw new Error(`审计日志写入失败: ${error.message}`);
    }
  }

  // 合规查询：按用户/时间范围导出日志
  async export(
    startDate: string,
    endDate: string,
    userId?: string,
  ): Promise<AuditEntry[]> {
    let query = this.supabase
      .from('audit_logs')
      .select('*')
      .gte('created_at', startDate)
      .lte('created_at', endDate);

    if (userId) {
      query = query.eq('user_id', userId);
    }

    const { data, error } = await query;
    if (error) throw new Error(`导出失败: ${error.message}`);
    return data as unknown as AuditEntry[];
  }
}
```

**真实案例参考：** GitLab 使用 Claude for Work 将内部文档和代码库接入 Claude，团队成员可以通过自然语言查询代码库、调试问题、学习新模块。关键是：**GitLab 的 IP 受到保护，Anthropic 不训练模型**。

---

## 23.3 场景二：代码审查自动化

这是开发团队最高频的使用场景。每次 Pull Request 提交后，自动触发 AI 代码审查，给出安全性、性能、规范方面的反馈。

**核心架构：**

```
GitHub/GitLab Webhook
  → 获取 PR diff
  → 分块（大 PR 拆成多个请求）
  → 并行调用 Claude API 审查各模块
  → 汇总结果
  →  posting comment 到 PR
  → 记录审查日志
```

### 23.3.1 完整实现

```typescript
// enterprise/code-review.ts
import Anthropic from '@anthropic-ai/sdk';

interface PRFile {
  filename: string;
  status: 'added' | 'modified' | 'deleted';
  additions: string;
  deletions: string;
  diff: string;
}

interface ReviewComment {
  file: string;
  line: number;
  severity: 'critical' | 'warning' | 'suggestion';
  category: 'security' | 'performance' | 'style' | 'logic' | 'best-practice';
  message: string;
  suggestion?: string;
}

interface CodeReviewResult {
  comments: ReviewComment[];
  summary: string;
  score: number; // 0-100，代码质量评分
}

// 企业代码审查机器人
class CodeReviewBot {
  private anthropic: Anthropic;
  private readonly MAX_DIFF_PER_REQUEST = 40000; // Token 预算

  constructor(apiKey: string) {
    this.anthropic = new Anthropic({ apiKey });
  }

  // 将大 PR 拆成可处理的块
  private chunkFiles(files: PRFile[]): PRFile[][] {
    const chunks: PRFile[][] = [];
    let currentChunk: PRFile[] = [];
    let currentSize = 0;

    for (const file of files) {
      const fileSize = file.diff.length;
      if (currentSize + fileSize > this.MAX_DIFF_PER_REQUEST && currentChunk.length > 0) {
        chunks.push(currentChunk);
        currentChunk = [];
        currentSize = 0;
      }
      currentChunk.push(file);
      currentSize += fileSize;
    }
    if (currentChunk.length > 0) {
      chunks.push(currentChunk);
    }
    return chunks;
  }

  // 审查单个代码块
  private async reviewChunk(
    files: PRFile[],
    focusAreas: string[],
  ): Promise<ReviewComment[]> {
    const diffText = files.map(f => {
      return `### 文件：${f.filename} (${f.status})
\`\`\`diff
${f.diff}
\`\`\`
`;
    }).join('\n');

    const focusPrompt = focusAreas.length > 0
      ? `重点关注以下方面：${focusAreas.join('、')}`
      : '全面审查代码质量';

    const response = await this.anthropic.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 4096,
      temperature: 0, // 代码审查需要确定性输出
      messages: [{
        role: 'user',
        content: `你是一个资深代码审查专家。请审查以下 Pull Request 的代码变更。

${focusPrompt}

## 审查标准：
1. **安全**：SQL 注入、XSS、权限绕过、敏感信息泄露
2. **性能**：N+1 查询、不必要的计算、内存泄漏风险
3. **规范**：命名规范、代码重复、函数过长
4. **逻辑**：边界条件、null 处理、并发安全
5. **最佳实践**：错误处理、日志记录、测试覆盖

## 输出格式（严格 JSON）：
{
  "comments": [
    {
      "file": "文件名",
      "line": 行号（估计）,
      "severity": "critical|warning|suggestion",
      "category": "security|performance|style|logic|best-practice",
      "message": "问题描述",
      "suggestion": "改进建议（可选）"
    }
  ]
}

## 代码变更：
${diffText}

只输出 JSON，不要加任何解释。`,
      }],
    });

    const content = response.content[0].type === 'text' ? response.content[0].text : '';
    try {
      const result = JSON.parse(content);
      return result.comments || [];
    } catch {
      // JSON 解析失败，尝试提取
      const jsonMatch = content.match(/\{[\s\S]*\}/);
      if (jsonMatch) {
        const result = JSON.parse(jsonMatch[0]);
        return result.comments || [];
      }
      return [];
    }
  }

  // 主审查入口（支持大 PR 并行审查）
  async reviewPR(
    files: PRFile[],
    options: {
      focusAreas?: string[];
      parallel?: boolean;
    } = {},
  ): Promise<CodeReviewResult> {
    const chunks = this.chunkFiles(files);

    // 并行审查各代码块
    const chunkResults = await Promise.all(
      chunks.map(chunk =>
        this.reviewChunk(chunk, options.focusAreas || [])
      )
    );

    const allComments = chunkResults.flat();

    // 生成总结
    const summary = await this.generateSummary(allComments, files);

    // 计算评分
    const score = this.calculateScore(allComments, files);

    return { comments: allComments, summary, score };
  }

  // 生成审查总结
  private async generateSummary(
    comments: ReviewComment[],
    files: PRFile[],
  ): Promise<string> {
    const stats = {
      total: comments.length,
      critical: comments.filter(c => c.severity === 'critical').length,
      warning: comments.filter(c => c.severity === 'warning').length,
      suggestion: comments.filter(c => c.severity === 'suggestion').length,
    };

    const response = await this.anthropic.messages.create({
      model: 'claude-haiku-4-20250514', // 总结用便宜的模型
      max_tokens: 1024,
      messages: [{
        role: 'user',
        content: `请为这次代码审查生成一个简洁的总结（中文，3-5句话）。

统计信息：
- 审查文件数：${files.length}
- 总问题数：${stats.total}
- 严重问题：${stats.critical}
- 警告：${stats.warning}
- 建议：${stats.suggestion}

问题详情：
${JSON.stringify(comments.map(c => ({ file: c.file, severity: c.severity, category: c.category, message: c.message })))}
`,
      }],
    });

    return response.content[0].type === 'text' ? response.content[0].text : '';
  }

  // 计算代码质量评分
  private calculateScore(comments: ReviewComment[], files: PRFile[]): number {
    if (files.length === 0) return 100;

    let penalty = 0;
    for (const c of comments) {
      switch (c.severity) {
        case 'critical': penalty += 15; break;
        case 'warning': penalty += 5; break;
        case 'suggestion': penalty += 1; break;
      }
    }

    const filePenalty = penalty / files.length;
    return Math.max(0, Math.min(100, 100 - filePenalty));
  }
}
```

### 23.3.2 与 GitHub PR 集成

```typescript
// enterprise/github-integration.ts
import { Octokit } from '@octokit/rest';

// 将审查结果 posting 到 GitHub PR
async function postReviewToPR(
  owner: string,
  repo: string,
  prNumber: number,
  result: CodeReviewResult,
  octokit: Octokit,
): Promise<void> {
  const { comments, summary, score } = result;

  // 1. 发表总结评论
  const scoreEmoji = score >= 80 ? '🟢' : score >= 60 ? '🟡' : '🔴';
  const summaryComment = `
## 🤖 AI 代码审查结果 ${scoreEmoji} ${score}/100

${summary}

### 问题统计
| 严重级别 | 数量 |
|---------|------|
| 🔴 严重 | ${comments.filter(c => c.severity === 'critical').length} |
| 🟡 警告 | ${comments.filter(c => c.severity === 'warning').length} |
| 🔵 建议 | ${comments.filter(c => c.severity === 'suggestion').length} |

---
> 由 Claude Code SDK 自动生成，仅供参考。
`;

  await octokit.issues.createComment({
    owner,
    repo,
    issue_number: prNumber,
    body: summaryComment,
  });

  // 2. 对严重问题，在具体代码行上添加 review comment
  for (const comment of comments.filter(c => c.severity === 'critical')) {
    // 注意：需要知道 diff 中的 position，这里简化为按文件名匹配
    await octokit.pulls.createReviewComment({
      owner,
      repo,
      pull_number: prNumber,
      body: `**${comment.category} 问题（严重）**\n\n${comment.message}\n\n${comment.suggestion ? `**建议：** ${comment.suggestion}` : ''}`,
      path: comment.file,
      line: comment.line,
      side: 'RIGHT',
    }).catch(err => {
      // 如果无法定位到具体行，降级为普通评论
      console.warn(`无法添加行内评论: ${err.message}`);
    });
  }
}
```

**真实案例参考：** Midjourney 使用 Claude 审查用户反馈、迭代审核策略，工程团队将 Claude 接入代码库做 troubleshooting。

---

## 23.4 场景三：客户支持智能化

Lyft 使用 Claude 将客户支持时间减少了 **87%**。怎么做到的？

**核心思路：** 把支持工单处理从「人工读 → 人工回」变成「AI 读 → AI 建议 → 人工确认」。

### 23.4.1 智能工单处理系统

```typescript
// enterprise/support-automation.ts
import Anthropic from '@anthropic-ai/sdk';

interface SupportTicket {
  id: string;
  customerId: string;
  priority: 'urgent' | 'high' | 'normal' | 'low';
  category: string;
  subject: string;
  description: string;
  history: { role: 'customer' | 'agent'; content: string; time: string }[];
  metadata: {
    planTier: string;    // 客户套餐等级
    language: string;    // 客户语言
    previousTickets: number; // 历史工单数
  };
}

interface TicketAnalysis {
  sentiment: 'angry' | 'frustrated' | 'neutral' | 'satisfied';
  intent: string;           // 用户真实意图
  category: string;         // 分类
  urgencyScore: number;     // 0-1，紧急程度
  suggestedReply: string;   // AI 建议回复
  internalNotes: string;    // 给客服的内部提示
  escalationNeeded: boolean; // 是否需要升级
}

// 客户支持智能化系统
class SupportIntelligence {
  private anthropic: Anthropic;
  private knowledgeBase: EnterpriseKnowledgeBase;

  constructor(
    anthropicApiKey: string,
    knowledgeBase: EnterpriseKnowledgeBase,
  ) {
    this.anthropic = new Anthropic({ apiKey: anthropicApiKey });
    this.knowledgeBase = knowledgeBase;
  }

  // 分析工单
  async analyzeTicket(ticket: SupportTicket): Promise<TicketAnalysis> {
    const conversationHistory = ticket.history.map(h =>
      `${h.role === 'customer' ? '客户' : '客服'}（${h.time}）：${h.content}`
    ).join('\n');

    const response = await this.anthropic.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 2048,
      messages: [{
        role: 'user',
        content: `你是一个客户支持分析专家。请分析以下支持工单，给出处理建议。

## 工单信息：
- ID：${ticket.id}
- 优先级：${ticket.priority}
- 客户等级：${ticket.metadata.planTier}
- 历史工单数：${ticket.metadata.previousTickets}

## 对话历史：
${conversationHistory}

## 当前问题：
${ticket.description}

## 输出格式（严格 JSON）：
{
  "sentiment": "angry|frustrated|neutral|satisfied",
  "intent": "用户真实意图（一句话）",
  "category": "billing|technical|feature-request|complaint|other",
  "urgencyScore": 0.0-1.0,
  "suggestedReply": "建议回复内容（专业、同理心）",
  "internalNotes": "给客服的内部提示（根因分析、注意事项）",
  "escalationNeeded": true/false
}

只输出 JSON。`,
      }],
    });

    const content = response.content[0].type === 'text' ? response.content[0].text : '';
    return JSON.parse(content.match(/\{[\s\S]*\}/)?.[0] || '{}');
  }

  // 生成个性化回复（结合知识库）
  async generateReply(
    ticket: SupportTicket,
    analysis: TicketAnalysis,
  ): Promise<string> {
    // 从知识库检索相关解决方案
    const kbResult = await this.knowledgeBase.ask(
      `如何处理：${analysis.intent}`,
      { id: 'system', roles: ['support'], department: 'support' },
    );

    const response = await this.anthropic.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 2048,
      messages: [{
        role: 'user',
        content: `你是一个专业的客户支持代表。请根据以下信息，生成一封回复。

## 客户问题：
${ticket.description}

## 知识库解决方案：
${kbResult.answer}

## 客户信息：
- 套餐等级：${ticket.metadata.planTier}
- 语言偏好：${ticket.metadata.language}
- 情绪状态：${analysis.sentiment}

## 回复要求：
1. 语气：${analysis.sentiment === 'angry' ? '非常抱歉，诚恳道歉' : '友好专业'}
2. 长度：简洁，不超过 200 字
3. 必须包含：具体解决步骤
4. 可选：如果知识库没有答案，提供替代方案并说明会转人工处理
5. 语言：${ticket.metadata.language === 'zh' ? '中文' : '英文'}

直接输出回复内容，不要加任何解释。`,
      }],
    });

    return response.content[0].type === 'text' ? response.content[0].text : '';
  }

  // 批量处理工单（优先级排序 + 并行处理）
  async processBatch(tickets: SupportTicket[]): Promise<Map<string, TicketAnalysis>> {
    // 1. 按紧急程度排序
    const sorted = [...tickets].sort((a, b) => {
      const priorityOrder = { urgent: 0, high: 1, normal: 2, low: 3 };
      return priorityOrder[a.priority] - priorityOrder[b.priority];
    });

    // 2. 并行分析（控制并发数，避免 API 限流）
    const results = new Map<string, TicketAnalysis>();
    const concurrency = 5;
    for (let i = 0; i < sorted.length; i += concurrency) {
      const batch = sorted.slice(i, i + concurrency);
      const batchResults = await Promise.all(
        batch.map(ticket => this.analyzeTicket(ticket).then(analysis => ({
          ticketId: ticket.id,
          analysis,
        })))
      );
      for (const { ticketId, analysis } of batchResults) {
        results.set(ticketId, analysis);
      }
    }

    return results;
  }
}
```

**真实数据：** Lyft 报告客户支持处理时间减少 87%；HubSpot 团队使用 Claude 后创造性工作时间增加。

---

## 23.5 场景四：多租户 SaaS 平台

如果你在做 ToB 的 SaaS 产品，每个客户的数据必须严格隔离，同时又要共享底层 AI 能力。

### 23.5.1 多租户架构设计

```typescript
// enterprise/multi-tenant.ts
import Anthropic from '@anthropic-ai/sdk';

interface Tenant {
  id: string;
  name: string;
  planTier: 'starter' | 'professional' | 'enterprise';
  apiKeyHash: string;      // API Key 的哈希（不存明文）
  rateLimitPerMinute: number;
  dataRegion: string;      // 数据存储区域（合规要求）
  features: string[];      // 启用的功能
}

interface TenantContext {
  tenant: Tenant;
  user: { id: string; roles: string[] };
  requestId: string;
  startTime: number;
}

// 多租户 AI 服务
class MultiTenantAIService {
  private anthropic: Anthropic;
  private tenantCache: Map<string, Tenant> = new Map();
  private rateLimiters: Map<string, RateLimiter> = new Map();

  constructor(apiKey: string) {
    this.anthropic = new Anthropic({ apiKey });
  }

  // 获取租户信息（带缓存）
  private async getTenant(tenantId: string): Promise<Tenant> {
    if (this.tenantCache.has(tenantId)) {
      return this.tenantCache.get(tenantId)!;
    }

    // 从数据库加载（示例）
    const tenant = await this.loadTenantFromDB(tenantId);
    this.tenantCache.set(tenantId, tenant);
    return tenant;
  }

  private async loadTenantFromDB(tenantId: string): Promise<Tenant> {
    // 实际实现：从数据库查询
    return {
      id: tenantId,
      name: `Tenant ${tenantId}`,
      planTier: 'professional',
      apiKeyHash: '',
      rateLimitPerMinute: 60,
      dataRegion: 'us-east-1',
      features: ['chat', 'embeddings'],
    };
  }

  // 获取租户的限流器
  private getRateLimiter(tenantId: string, limit: number): RateLimiter {
    if (!this.rateLimiter.has(tenantId)) {
      this.rateLimiters.set(tenantId, new RateLimiter(limit));
    }
    return this.rateLimiters.get(tenantId)!;
  }

  // 带租户隔离的 API 调用
  async chat(
    tenantId: string,
    user: TenantContext['user'],
    messages: Anthropic.Messages.MessageParam[],
    options: { model?: string; maxTokens?: number } = {},
  ): Promise<Anthropic.Messages.Message> {
    const tenant = await this.getTenant(tenantId);
    const ctx: TenantContext = {
      tenant,
      user,
      requestId: crypto.randomUUID(),
      startTime: Date.now(),
    };

    // 1. 权限检查
    this.checkPermissions(ctx, 'chat');

    // 2. 速率限制
    const limiter = this.getRateLimiter(tenantId, tenant.rateLimitPerMinute);
    await limiter.acquire();

    // 3. 数据隔离：每个租户的 conversation 用独立 namespace
    const namespacedMessages = messages.map(m => ({
      ...m,
      // 在实际实现中，这里可以加密或添加租户标记
    }));

    // 4. 模型选择（按套餐限制）
    const model = this.resolveModel(options.model, tenant.planTier);

    try {
      // 5. 调用 API
      const response = await this.anthropic.messages.create({
        model,
        max_tokens: options.maxTokens || 4096,
        messages: namespacedMessages,
      });

      // 6. 记录用量（按租户计量）
      await this.recordUsage(ctx, {
        inputTokens: response.usage.input_tokens,
        outputTokens: response.usage.output_tokens,
        model,
      });

      return response;
    } catch (error) {
      // 7. 错误日志（包含租户信息，但不泄露敏感数据）
      console.error(`[Tenant: ${tenantId}] API 调用失败:`, {
        requestId: ctx.requestId,
        error: error instanceof Error ? error.message : 'Unknown error',
      });
      throw error;
    }
  }

  // 根据租户套餐解析模型
  private resolveModel(requested: string | undefined, tier: string): string {
    const modelMap: Record<string, string[]> = {
      starter: ['claude-haiku-4-20250514'],
      professional: ['claude-haiku-4-20250514', 'claude-sonnet-4-20250514'],
      enterprise: ['claude-haiku-4-20250514', 'claude-sonnet-4-20250514', 'claude-opus-4-20250514'],
    };

    const allowed = modelMap[tier] || modelMap.starter;
    if (requested && allowed.includes(requested)) return requested;
    // 默认使用套餐允许的最强模型
    return allowed[allowed.length - 1];
  }

  // 权限检查
  private checkPermissions(ctx: TenantContext, feature: string): void {
    if (!ctx.tenant.features.includes(feature)) {
      throw new Error(`租户 ${ctx.tenant.id} 未启用功能：${feature}`);
    }
  }

  // 记录用量（用于计费）
  private async recordUsage(
    ctx: TenantContext,
    usage: { inputTokens: number; outputTokens: number; model: string },
  ): Promise<void> {
    const pricing: Record<string, { input: number; output: number }> = {
      'claude-opus-4-20250514':    { input: 15.0, output: 75.0 },
      'claude-sonnet-4-20250514':   { input: 3.0,  output: 15.0 },
      'claude-haiku-4-20250514':    { input: 0.25, output: 1.25 },
    };

    const p = pricing[usage.model] || { input: 0, output: 0 };
    const cost = (usage.inputTokens / 1_000_000) * p.input
               + (usage.outputTokens / 1_000_000) * p.output;

    // 写入用量记录（实际实现：写入数据库）
    console.log(`[Usage] tenant=${ctx.tenant.id} cost=$${cost.toFixed(4)} tokens=${usage.inputTokens + usage.outputTokens}`);
  }
}

// 简单的速率限制器（令牌桶）
class RateLimiter {
  private tokens: number;
  private lastRefill: number;
  private readonly maxTokens: number;
  private readonly refillRate: number; // tokens per ms

  constructor(requestsPerMinute: number) {
    this.maxTokens = requestsPerMinute;
    this.tokens = requestsPerMinute;
    this.lastRefill = Date.now();
    this.refillRate = requestsPerMinute / 60000; // per ms
  }

  async acquire(): Promise<void> {
    this.refill();
    if (this.tokens >= 1) {
      this.tokens -= 1;
      return;
    }
    // 等待直到有令牌
    const waitMs = (1 - this.tokens) / this.refillRate;
    await new Promise(resolve => setTimeout(resolve, waitMs));
    this.tokens = 0;
  }

  private refill(): void {
    const now = Date.now();
    const elapsed = now - this.lastRefill;
    this.tokens = Math.min(this.maxTokens, this.tokens + elapsed * this.refillRate);
    this.lastRefill = now;
  }
}
```

---

## 23.6 企业级最佳实践总结

把这一章的内容浓缩成一张检查清单：

### 安全与合规
- ✅ **数据隔离**：多租户数据物理/逻辑隔离
- ✅ **审计日志**：每次 API 调用都有记录，不可篡改
- ✅ **SSO 集成**：不支持账号密码登录（用 SAML/OAuth）
- ✅ **数据不用于训练**：与 Anthropic 签署数据处理协议
- ✅ **敏感信息过滤**：发送前脱敏（PII、API Key、密码）

### 可靠性
- ✅ **降级策略**：API 不可用时有本地回退方案
- ✅ **速率限制**：保护自己的 API 不被客户打垮
- ✅ **超时控制**：所有 API 调用设合理超时
- ✅ **重试机制**：指数退避 + 断路器（第21章）

### 成本
- ✅ **按租户计量**：知道每个客户花了多少钱
- ✅ **模型降级**：starter 套餐只能用 Haiku
- ✅ **Token 预算**：每次请求设 max_tokens 上限
- ✅ **缓存**：相同问题不重复调用 API

### 监控
- ✅ **实时仪表盘**：Token 消耗、错误率、延迟
- ✅ **成本预警**：超预算自动告警
- ✅ **审计导出**：支持合规审计的日志导出

---

## 23.7 本章小结

这一章我们覆盖了四个真实的企业级应用场景：

1. **知识库问答** — 向量检索 + 权限过滤 + 审计日志
2. **代码审查自动化** — 大 PR 分块 + 并行处理 + GitHub 集成
3. **客户支持智能化** — 情感分析 + 知识库联动 + 批量处理
4. **多租户 SaaS** — 数据隔离 + 速率限制 + 按租户计费

这些场景的代码都不是 Demo 级别——它们考虑了权限、审计、限流、降级、计费，是企业生产环境真正需要的架构。

**下一章**我们会讲 Rate Limiting 与配额管理，这是企业级应用的另一块基石。

> "企业级"不是一句口号，而是一堆细节堆出来的。
