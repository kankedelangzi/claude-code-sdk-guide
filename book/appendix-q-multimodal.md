# 附录Q：Claude Code SDK 多模态应用实战

> "多模态不是未来，而是当下。图像、语音、文本——三者合一，才是真正的智能。" —— 老三

Claude 从一开始就支持多模态能力。本章深入讲解如何在 Claude Code SDK 中构建多模态应用——图像理解、语音处理、以及文本+图像混合的复杂场景。

---

## Q.1 多模态能力概览

### Q.1.1 Claude 支持的多模态类型

| 模态 | 输入 | 输出 | 支持状态 |
|-----|------|------|---------|
| 文本 | ✅ | ✅ | 核心能力 |
| 图像 | ✅ | ❌ | 支持识别与分析 |
| 音频 | ⚠️ | ⚠️ | 需第三方 API 配合 |
| 视频 | ⚠️ | ❌ | 需帧提取 + 图像处理 |

**说明：**
- ✅ 原生支持
- ⚠️ 需要额外处理或第三方服务
- ❌ 不支持

### Q.1.2 多模态应用场景矩阵

```
┌─────────────────────────────────────────────────────────────┐
│                    多模态应用场景                            │
├───────────────────┬───────────────────┬─────────────────────┤
│      图像理解      │      语音处理      │      混合应用        │
├───────────────────┼───────────────────┼─────────────────────┤
│ • 截图分析         │ • 语音转文字(STT)   │ • 图文问答           │
│ • UI/UX 评审      │ • 文字转语音(TTS)   │ • 文档 OCR          │
│ • 图表解读         │ • 语音助手         │ • 多模态搜索         │
│ • 医学影像辅助     │ • 会议录音摘要     │ • 内容审核           │
│ • 产品图片描述     │ • 播客转录         │ • 智能相册          │
└───────────────────┴───────────────────┴─────────────────────┘
```

---

## Q.2 图像处理实战

### Q.2.1 图像输入的两种方式

Claude Code SDK 支持两种图像输入方式：

**方式一：Base64 编码**

```typescript
import { ClaudeCode } from '@anthropic-ai/claude-code-sdk';
import * as fs from 'fs';

const claude = new ClaudeCode();

async function analyzeLocalImage(imagePath: string) {
  // 读取图片并转为 Base64
  const imageBuffer = fs.readFileSync(imagePath);
  const base64Data = imageBuffer.toString('base64');
  
  // 推断 MIME 类型
  const ext = imagePath.split('.').pop()?.toLowerCase();
  const mimeTypes: Record<string, string> = {
    'jpg': 'image/jpeg',
    'jpeg': 'image/jpeg',
    'png': 'image/png',
    'gif': 'image/gif',
    'webp': 'image/webp',
  };
  const mediaType = mimeTypes[ext || ''] || 'image/png';

  const response = await claude.messages.create({
    model: 'claude-sonnet-4-20250514',
    maxTokens: 2048,
    messages: [
      {
        role: 'user',
        content: [
          {
            type: 'image',
            source: {
              type: 'base64',
              media_type: mediaType as any,
              data: base64Data,
            },
          },
          {
            type: 'text',
            text: '请详细描述这张图片的内容，包括主体、背景、色彩和可能的拍摄意图。',
          },
        ],
      },
    ],
  });

  return response.content
    .filter(block => block.type === 'text')
    .map(block => (block as any).text)
    .join('\n');
}

// 使用
analyzeLocalImage('./photos/sunset.jpg').then(console.log);
```

**方式二：URL 引用**

```typescript
async function analyzeImageFromUrl(imageUrl: string, prompt: string) {
  const response = await claude.messages.create({
    model: 'claude-sonnet-4-20250514',
    maxTokens: 2048,
    messages: [
      {
        role: 'user',
        content: [
          {
            type: 'image',
            source: {
              type: 'url',
              url: imageUrl,
            },
          },
          {
            type: 'text',
            text: prompt,
          },
        ],
      },
    ],
  });

  return response.content
    .filter(block => block.type === 'text')
    .map(block => (block as any).text)
    .join('\n');
}

// 使用
analyzeImageFromUrl(
  'https://example.com/chart.png',
  '分析这个图表的趋势，并给出数据解读。'
).then(console.log);
```

### Q.2.2 图像预处理：压缩与优化

大图片会消耗更多 Token 并增加响应延迟。预处理是生产环境的必备环节。

```typescript
import sharp from 'sharp';

interface ImageProcessOptions {
  maxWidth?: number;
  maxHeight?: number;
  quality?: number;
  format?: 'jpeg' | 'png' | 'webp';
}

async function processImage(
  inputPath: string,
  options: ImageProcessOptions = {}
): Promise<{ data: string; mediaType: string }> {
  const {
    maxWidth = 1600,
    maxHeight = 1600,
    quality = 85,
    format = 'jpeg',
  } = options;

  const image = sharp(inputPath);
  const metadata = await image.metadata();

  // 如果图片过大，进行缩放
  if (metadata.width && metadata.width > maxWidth) {
    image.resize(maxWidth, maxHeight, {
      fit: 'inside',
      withoutEnlargement: true,
    });
  }

  // 转换格式并压缩
  let processedBuffer: Buffer;
  let mediaType: string;

  switch (format) {
    case 'jpeg':
      processedBuffer = await image.jpeg({ quality }).toBuffer();
      mediaType = 'image/jpeg';
      break;
    case 'png':
      processedBuffer = await image.png({ compressionLevel: 9 }).toBuffer();
      mediaType = 'image/png';
      break;
    case 'webp':
      processedBuffer = await image.webp({ quality }).toBuffer();
      mediaType = 'image/webp';
      break;
    default:
      processedBuffer = await image.jpeg({ quality }).toBuffer();
      mediaType = 'image/jpeg';
  }

  return {
    data: processedBuffer.toString('base64'),
    mediaType,
  };
}

// 完整流程：预处理 + 分析
async function analyzeWithPreprocessing(imagePath: string) {
  const { data, mediaType } = await processImage(imagePath, {
    maxWidth: 1200,
    quality: 80,
    format: 'jpeg',
  });

  const response = await claude.messages.create({
    model: 'claude-sonnet-4-20250514',
    maxTokens: 2048,
    messages: [
      {
        role: 'user',
        content: [
          { type: 'image', source: { type: 'base64', media_type: mediaType as any, data } },
          { type: 'text', text: '描述这张图片' },
        ],
      },
    ],
  });

  return response.content.filter(b => b.type === 'text').map(b => (b as any).text).join('');
}
```

### Q.2.3 实战案例：UI 截图评审助手

一个实用的多模态应用——自动评审 UI 截图，指出可用性问题。

```typescript
#!/usr/bin/env node
import { ClaudeCode } from '@anthropic-ai/claude-code-sdk';
import * as fs from 'fs';
import * as path from 'path';
import sharp from 'sharp';

const claude = new ClaudeCode();

interface UIReviewResult {
  overallScore: number;
  issues: Array<{
    severity: 'high' | 'medium' | 'low';
    category: string;
    description: string;
    suggestion: string;
  }>;
  positives: string[];
}

async function reviewUIScreenshot(imagePath: string): Promise<UIReviewResult> {
  // 预处理图片
  const { data, mediaType } = await processImage(imagePath);

  const prompt = `
你是一位资深的 UI/UX 专家。请评审这个界面截图，从以下维度分析：

1. **布局与层次** - 信息架构是否清晰
2. **视觉设计** - 颜色、字体、间距是否合理
3. **交互设计** - 按钮、表单、导航是否易用
4. **可访问性** - 对色盲、视障用户是否友好
5. **一致性** - 设计语言是否统一

请以 JSON 格式输出评审结果：
{
  "overallScore": 1-10 分,
  "issues": [
    {
      "severity": "high/medium/low",
      "category": "分类",
      "description": "问题描述",
      "suggestion": "改进建议"
    }
  ],
  "positives": ["做得好的地方"]
}
`;

  const response = await claude.messages.create({
    model: 'claude-sonnet-4-20250514',
    maxTokens: 4096,
    messages: [
      {
        role: 'user',
        content: [
          { type: 'image', source: { type: 'base64', media_type: mediaType as any, data } },
          { type: 'text', text: prompt },
        ],
      },
    ],
  });

  const textContent = response.content
    .filter(b => b.type === 'text')
    .map(b => (b as any).text)
    .join('');

  // 提取 JSON
  const jsonMatch = textContent.match(/\{[\s\S]*\}/);
  if (jsonMatch) {
    return JSON.parse(jsonMatch[0]);
  }

  throw new Error('无法解析评审结果');
}

// 批量评审
async function batchReviewUI(directory: string) {
  const files = fs.readdirSync(directory)
    .filter(f => /\.(jpg|jpeg|png|webp)$/i.test(f));

  const results: Array<{ file: string; review: UIReviewResult }> = [];

  for (const file of files) {
    const filePath = path.join(directory, file);
    console.log(`评审: ${file}...`);
    
    try {
      const review = await reviewUIScreenshot(filePath);
      results.push({ file, review });
      console.log(`  得分: ${review.overallScore}/10`);
    } catch (error) {
      console.error(`  错误: ${error}`);
    }
  }

  // 生成报告
  const avgScore = results.reduce((sum, r) => sum + r.review.overallScore, 0) / results.length;
  console.log(`\n=== 评审报告 ===`);
  console.log(`总截图数: ${results.length}`);
  console.log(`平均得分: ${avgScore.toFixed(1)}/10`);

  return results;
}

// CLI 入口
if (process.argv[2]) {
  reviewUIScreenshot(process.argv[2])
    .then(result => console.log(JSON.stringify(result, null, 2)))
    .catch(console.error);
}
```

**使用示例：**

```bash
# 单张截图评审
tsx ui-reviewer.ts ./screenshots/home.png

# 批量评审
tsx ui-reviewer.ts ./screenshots/
```

### Q.2.4 实战案例：文档 OCR 与结构化提取

将图片中的文档内容提取为结构化数据。

```typescript
interface DocumentExtraction {
  type: 'invoice' | 'receipt' | 'contract' | 'form' | 'other';
  fields: Record<string, string>;
  rawText: string;
  confidence: number;
}

async function extractDocument(imagePath: string): Promise<DocumentExtraction> {
  const { data, mediaType } = await processImage(imagePath);

  const prompt = `
请分析这张文档图片，执行以下任务：

1. **识别文档类型**：发票(invoice)、收据(receipt)、合同(contract)、表单(form)或其他(other)
2. **提取关键字段**：根据文档类型提取关键字段（如发票号、日期、金额等）
3. **提取全部文本**：按原始布局提取所有文字内容

以 JSON 格式返回：
{
  "type": "文档类型",
  "fields": { "字段名": "值" },
  "rawText": "完整文本内容",
  "confidence": 0-1 之间的置信度
}
`;

  const response = await claude.messages.create({
    model: 'claude-sonnet-4-20250514',
    maxTokens: 4096,
    messages: [
      {
        role: 'user',
        content: [
          { type: 'image', source: { type: 'base64', media_type: mediaType as any, data } },
          { type: 'text', text: prompt },
        ],
      },
    ],
  });

  const textContent = response.content
    .filter(b => b.type === 'text')
    .map(b => (b as any).text)
    .join('');

  const jsonMatch = textContent.match(/\{[\s\S]*\}/);
  if (jsonMatch) {
    return JSON.parse(jsonMatch[0]);
  }

  throw new Error('无法解析文档提取结果');
}

// 使用示例
extractDocument('./documents/invoice-001.jpg')
  .then(result => {
    console.log('文档类型:', result.type);
    console.log('提取字段:', result.fields);
    console.log('置信度:', result.confidence);
  });
```

---

## Q.3 多图处理与比较

### Q.3.1 多图输入

Claude 支持在一次请求中发送多张图片，用于比较、组合分析。

```typescript
async function compareImages(imagePaths: string[], question: string) {
  // 预处理所有图片
  const imageBlocks = await Promise.all(
    imagePaths.map(async (path) => {
      const { data, mediaType } = await processImage(path);
      return {
        type: 'image' as const,
        source: { type: 'base64' as const, media_type: mediaType as any, data },
      };
    })
  );

  const response = await claude.messages.create({
    model: 'claude-sonnet-4-20250514',
    maxTokens: 4096,
    messages: [
      {
        role: 'user',
        content: [
          ...imageBlocks,
          { type: 'text', text: question },
        ],
      },
    ],
  });

  return response.content.filter(b => b.type === 'text').map(b => (b as any).text).join('');
}

// 使用：比较两个版本的 UI 设计
compareImages(
  ['./design-v1.png', './design-v2.png'],
  '比较这两个 UI 设计版本，列出主要的视觉差异，并推荐更好的版本。'
).then(console.log);
```

### Q.3.2 实战案例：A/B 测试结果分析

```typescript
interface ABTestAnalysis {
  versionA: { strengths: string[]; weaknesses: string[] };
  versionB: { strengths: string[]; weaknesses: string[] };
  recommendation: 'A' | 'B' | 'tie';
  reasoning: string;
}

async function analyzeABTest(
  imageA: string,
  imageB: string,
  testContext: string
): Promise<ABTestAnalysis> {
  const [imgA, imgB] = await Promise.all([
    processImage(imageA),
    processImage(imageB),
  ]);

  const prompt = `
你是一位 A/B 测试专家。请分析这两个设计版本：

测试背景：${testContext}

请评估：
1. 版本 A 的优势和劣势
2. 版本 B 的优势和劣势
3. 哪个版本更可能胜出（考虑点击率、转化率等指标）

以 JSON 格式返回：
{
  "versionA": { "strengths": [], "weaknesses": [] },
  "versionB": { "strengths": [], "weaknesses": [] },
  "recommendation": "A" | "B" | "tie",
  "reasoning": "推荐理由"
}
`;

  const response = await claude.messages.create({
    model: 'claude-sonnet-4-20250514',
    maxTokens: 4096,
    messages: [
      {
        role: 'user',
        content: [
          { type: 'image', source: { type: 'base64', media_type: imgA.mediaType as any, data: imgA.data } },
          { type: 'image', source: { type: 'base64', media_type: imgB.mediaType as any, data: imgB.data } },
          { type: 'text', text: prompt },
        ],
      },
    ],
  });

  const textContent = response.content.filter(b => b.type === 'text').map(b => (b as any).text).join('');
  const jsonMatch = textContent.match(/\{[\s\S]*\}/);
  
  if (jsonMatch) {
    return JSON.parse(jsonMatch[0]);
  }

  throw new Error('无法解析 A/B 测试分析结果');
}
```

---

## Q.4 语音处理实战

虽然 Claude 本身不直接处理音频，但可以通过集成第三方 API 实现完整的语音应用。

### Q.4.1 语音转文字（STT）+ Claude 分析

```typescript
import Anthropic from '@anthropic-ai/sdk';
import FormData from 'form-data';
import fetch from 'node-fetch';

// 配置 STT 服务（以 OpenAI Whisper API 为例）
const OPENAI_API_KEY = process.env.OPENAI_API_KEY;

async function transcribeAudio(audioPath: string): Promise<string> {
  const formData = new FormData();
  formData.append('file', require('fs').createReadStream(audioPath));
  formData.append('model', 'whisper-1');

  const response = await fetch('https://api.openai.com/v1/audio/transcriptions', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${OPENAI_API_KEY}`,
      ...formData.getHeaders(),
    },
    body: formData,
  });

  const result = await response.json() as any;
  return result.text;
}

// 完整流程：语音 → 文字 → Claude 分析
async function processVoiceCommand(audioPath: string): Promise<string> {
  // Step 1: 语音转文字
  console.log('转录中...');
  const transcript = await transcribeAudio(audioPath);
  console.log('转录结果:', transcript);

  // Step 2: Claude 理解并处理
  const response = await claude.messages.create({
    model: 'claude-sonnet-4-20250514',
    maxTokens: 2048,
    system: '你是一个语音助手。用户的语音已转录为文字，请理解意图并给出回应。',
    messages: [
      { role: 'user', content: transcript },
    ],
  });

  return response.content.filter(b => b.type === 'text').map(b => (b as any).text).join('');
}

// 使用
processVoiceCommand('./recordings/command.mp3').then(console.log);
```

### Q.4.2 实战案例：会议录音智能摘要

```typescript
interface MeetingSummary {
  title: string;
  date: string;
  participants: string[];
  keyPoints: string[];
  actionItems: Array<{ task: string; owner: string; dueDate?: string }>;
  decisions: string[];
}

async function summarizeMeeting(audioPath: string): Promise<MeetingSummary> {
  // 转录音频
  const transcript = await transcribeAudio(audioPath);

  // Claude 生成结构化摘要
  const prompt = `
请根据以下会议转录内容生成结构化摘要：

${transcript}

以 JSON 格式返回：
{
  "title": "会议主题",
  "date": "日期（如能识别）",
  "participants": ["参会人员"],
  "keyPoints": ["关键讨论点"],
  "actionItems": [{ "task": "待办事项", "owner": "负责人", "dueDate": "截止日期" }],
  "decisions": ["会议决议"]
}
`;

  const response = await claude.messages.create({
    model: 'claude-sonnet-4-20250514',
    maxTokens: 4096,
    messages: [{ role: 'user', content: prompt }],
  });

  const textContent = response.content.filter(b => b.type === 'text').map(b => (b as any).text).join('');
  const jsonMatch = textContent.match(/\{[\s\S]*\}/);
  
  if (jsonMatch) {
    return JSON.parse(jsonMatch[0]);
  }

  throw new Error('无法解析会议摘要');
}
```

### Q.4.3 文字转语音（TTS）输出

将 Claude 的回复转为语音输出。

```typescript
async function speakResponse(text: string, outputPath: string): Promise<void> {
  const response = await fetch('https://api.openai.com/v1/audio/speech', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${OPENAI_API_KEY}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      model: 'tts-1',
      input: text,
      voice: 'nova',
    }),
  });

  const buffer = await response.buffer();
  require('fs').writeFileSync(outputPath, buffer);
}

// 完整语音对话流程
async function voiceConversation(audioInput: string): Promise<string> {
  // 1. 语音输入 → 文字
  const userText = await transcribeAudio(audioInput);
  console.log('用户:', userText);

  // 2. Claude 处理
  const response = await claude.messages.create({
    model: 'claude-sonnet-4-20250514',
    maxTokens: 1024,
    messages: [{ role: 'user', content: userText }],
  });

  const assistantText = response.content
    .filter(b => b.type === 'text')
    .map(b => (b as any).text)
    .join('');

  // 3. 文字 → 语音输出
  const audioOutput = './output.mp3';
  await speakResponse(assistantText, audioOutput);
  
  console.log('助手:', assistantText);
  console.log('语音输出已保存:', audioOutput);

  return assistantText;
}
```

---

## Q.5 图像 + 文本混合应用

### Q.5.1 图文问答系统

结合图像理解和文本问答，实现更智能的交互。

```typescript
interface VisualQA {
  answer: string;
  confidence: number;
  relatedQuestions: string[];
}

async function visualQuestionAnswering(
  imagePath: string,
  question: string
): Promise<VisualQA> {
  const { data, mediaType } = await processImage(imagePath);

  const response = await claude.messages.create({
    model: 'claude-sonnet-4-20250514',
    maxTokens: 2048,
    messages: [
      {
        role: 'user',
        content: [
          { type: 'image', source: { type: 'base64', media_type: mediaType as any, data } },
          { type: 'text', text: question },
        ],
      },
    ],
  });

  const answer = response.content
    .filter(b => b.type === 'text')
    .map(b => (b as any).text)
    .join('');

  // 生成相关问题
  const followUp = await claude.messages.create({
    model: 'claude-sonnet-4-20250514',
    maxTokens: 512,
    messages: [
      {
        role: 'user',
        content: `基于这个图片问答，生成3个相关的后续问题：

图片问题：${question}
回答：${answer}

以 JSON 数组格式返回：["问题1", "问题2", "问题3"]`,
      },
    ],
  });

  const relatedText = followUp.content
    .filter(b => b.type === 'text')
    .map(b => (b as any).text)
    .join('');

  const jsonMatch = relatedText.match(/\[[\s\S]*\]/);

  return {
    answer,
    confidence: 0.85, // 简化处理
    relatedQuestions: jsonMatch ? JSON.parse(jsonMatch[0]) : [],
  };
}
```

### Q.5.2 实战案例：智能相册管理

```typescript
interface PhotoMetadata {
  description: string;
  tags: string[];
  people: string[];
  location?: string;
  mood: string;
  quality: 'excellent' | 'good' | 'fair' | 'poor';
  suggestedAlbums: string[];
}

async function analyzePhoto(imagePath: string): Promise<PhotoMetadata> {
  const { data, mediaType } = await processImage(imagePath, { maxWidth: 800 });

  const prompt = `
分析这张照片，提取以下信息：

以 JSON 格式返回：
{
  "description": "一句话描述照片内容",
  "tags": ["标签1", "标签2"],
  "people": ["识别到的人物描述"],
  "location": "可能的拍摄地点（如能判断）",
  "mood": "照片氛围（如温馨、欢乐、宁静等）",
  "quality": "excellent/good/fair/poor",
  "suggestedAlbums": ["建议的分类相册"]
}
`;

  const response = await claude.messages.create({
    model: 'claude-sonnet-4-20250514',
    maxTokens: 1024,
    messages: [
      {
        role: 'user',
        content: [
          { type: 'image', source: { type: 'base64', media_type: mediaType as any, data } },
          { type: 'text', text: prompt },
        ],
      },
    ],
  });

  const textContent = response.content
    .filter(b => b.type === 'text')
    .map(b => (b as any).text)
    .join('');

  const jsonMatch = textContent.match(/\{[\s\S]*\}/);
  if (jsonMatch) {
    return JSON.parse(jsonMatch[0]);
  }

  throw new Error('无法解析照片元数据');
}

// 批量处理相册
async function organizePhotoAlbum(directory: string) {
  const files = require('fs')
    .readdirSync(directory)
    .filter((f: string) => /\.(jpg|jpeg|png|webp)$/i.test(f));

  const photoData: Array<{ file: string; metadata: PhotoMetadata }> = [];

  for (const file of files) {
    const filePath = path.join(directory, file);
    console.log(`分析: ${file}...`);

    try {
      const metadata = await analyzePhoto(filePath);
      photoData.push({ file, metadata });
      console.log(`  描述: ${metadata.description}`);
      console.log(`  标签: ${metadata.tags.join(', ')}`);
    } catch (error) {
      console.error(`  错误: ${error}`);
    }
  }

  // 按标签分组
  const albumGroups: Record<string, string[]> = {};
  for (const { file, metadata } of photoData) {
    for (const album of metadata.suggestedAlbums) {
      if (!albumGroups[album]) {
        albumGroups[album] = [];
      }
      albumGroups[album].push(file);
    }
  }

  console.log('\n=== 相册分组建议 ===');
  for (const [album, files] of Object.entries(albumGroups)) {
    console.log(`${album}: ${files.length} 张照片`);
  }

  return { photoData, albumGroups };
}
```

---

## Q.6 视频处理实战

视频处理需要先将视频转为帧序列，然后批量处理图像。

### Q.6.1 视频帧提取 + 分析

```typescript
import { exec } from 'child_process';
import { promisify } from 'util';

const execAsync = promisify(exec);

async function extractFrames(
  videoPath: string,
  outputDir: string,
  fps: number = 1
): Promise<string[]> {
  // 使用 ffmpeg 提取帧
  const outputPattern = `${outputDir}/frame_%04d.jpg`;
  await execAsync(`ffmpeg -i "${videoPath}" -vf fps=${fps} "${outputPattern}"`);

  // 返回帧文件列表
  return require('fs')
    .readdirSync(outputDir)
    .filter((f: string) => f.startsWith('frame_'))
    .sort()
    .map((f: string) => path.join(outputDir, f));
}

async function analyzeVideoKeyframes(videoPath: string): Promise<string[]> {
  const frameDir = './temp_frames';
  require('fs').mkdirSync(frameDir, { recursive: true });

  // 提取每秒 1 帧
  const frames = await extractFrames(videoPath, frameDir, 1);
  console.log(`提取了 ${frames.length} 帧`);

  // 分析关键帧
  const analyses: string[] = [];

  // 每隔 5 帧采样一次（降低成本）
  const sampleFrames = frames.filter((_, i) => i % 5 === 0);

  for (const frame of sampleFrames) {
    const { data, mediaType } = await processImage(frame, { maxWidth: 800 });

    const response = await claude.messages.create({
      model: 'claude-sonnet-4-20250514',
      maxTokens: 256,
      messages: [
        {
          role: 'user',
          content: [
            { type: 'image', source: { type: 'base64', media_type: mediaType as any, data } },
            { type: 'text', text: '用一句话描述这帧画面内容。' },
          ],
        },
      ],
    });

    const description = response.content
      .filter(b => b.type === 'text')
      .map(b => (b as any).text)
      .join('');

    analyses.push(description);
  }

  // 清理临时文件
  frames.forEach(f => require('fs').unlinkSync(f));
  require('fs').rmdirSync(frameDir);

  return analyses;
}

// 视频摘要
async function summarizeVideo(videoPath: string): Promise<string> {
  const frameAnalyses = await analyzeVideoKeyframes(videoPath);

  const response = await claude.messages.create({
    model: 'claude-sonnet-4-20250514',
    maxTokens: 1024,
    messages: [
      {
        role: 'user',
        content: `以下是视频关键帧的描述，请生成视频摘要：

${frameAnalyses.map((a, i) => `帧${i + 1}: ${a}`).join('\n')}`,
      },
    ],
  });

  return response.content
    .filter(b => b.type === 'text')
    .map(b => (b as any).text)
    .join('');
}
```

### Q.6.2 实战案例：短视频内容审核

```typescript
interface VideoAuditResult {
  overallVerdict: 'pass' | 'warning' | 'reject';
  issues: Array<{
    timestamp: string;
    type: string;
    severity: 'high' | 'medium' | 'low';
    description: string;
  }>;
  confidence: number;
}

async function auditVideo(videoPath: string): Promise<VideoAuditResult> {
  const frameDir = './audit_frames';
  require('fs').mkdirSync(frameDir, { recursive: true });

  // 提取帧（每 0.5 秒一帧）
  const frames = await extractFrames(videoPath, frameDir, 2);

  const issues: VideoAuditResult['issues'] = [];

  for (let i = 0; i < frames.length; i++) {
    const frame = frames[i];
    const { data, mediaType } = await processImage(frame, { maxWidth: 600 });

    const response = await claude.messages.create({
      model: 'claude-sonnet-4-20250514',
      maxTokens: 512,
      messages: [
        {
          role: 'user',
          content: [
            { type: 'image', source: { type: 'base64', media_type: mediaType as any, data } },
            {
              type: 'text',
              text: `检查这帧画面是否有以下违规内容：
1. 色情/裸露内容
2. 暴力/血腥内容
3. 违禁品/危险行为
4. 不当广告/水印

以 JSON 格式返回：
{
  "hasViolation": true/false,
  "violations": [{ "type": "类型", "severity": "high/medium/low", "description": "描述" }]
}`,
            },
          ],
        },
      ],
    });

    const textContent = response.content
      .filter(b => b.type === 'text')
      .map(b => (b as any).text)
      .join('');

    const jsonMatch = textContent.match(/\{[\s\S]*\}/);
    if (jsonMatch) {
      const result = JSON.parse(jsonMatch[0]);
      if (result.hasViolation) {
        for (const v of result.violations) {
          issues.push({
            timestamp: `${(i * 0.5).toFixed(1)}s`,
            ...v,
          });
        }
      }
    }
  }

  // 清理
  frames.forEach(f => require('fs').unlinkSync(f));
  require('fs').rmdirSync(frameDir);

  // 判断整体结果
  const hasHighSeverity = issues.some(i => i.severity === 'high');
  const hasMediumSeverity = issues.some(i => i.severity === 'medium');

  let overallVerdict: 'pass' | 'warning' | 'reject';
  if (hasHighSeverity) {
    overallVerdict = 'reject';
  } else if (hasMediumSeverity) {
    overallVerdict = 'warning';
  } else {
    overallVerdict = 'pass';
  }

  return {
    overallVerdict,
    issues,
    confidence: 0.9,
  };
}
```

---

## Q.7 多模态 Agent 架构

### Q.7.1 统一多模态处理器

```typescript
type MediaType = 'text' | 'image' | 'audio' | 'video';

interface MultimodalRequest {
  type: MediaType;
  content: string | Buffer;
  prompt?: string;
}

interface MultimodalResponse {
  analysis: string;
  structuredData?: any;
  suggestions?: string[];
}

class MultimodalAgent {
  private claude: ClaudeCode;

  constructor() {
    this.claude = new ClaudeCode();
  }

  async process(request: MultimodalRequest): Promise<MultimodalResponse> {
    switch (request.type) {
      case 'text':
        return this.processText(request.content as string, request.prompt);
      case 'image':
        return this.processImage(request.content as Buffer, request.prompt);
      case 'audio':
        return this.processAudio(request.content as Buffer, request.prompt);
      case 'video':
        return this.processVideo(request.content as string, request.prompt);
      default:
        throw new Error(`不支持的媒体类型: ${request.type}`);
    }
  }

  private async processText(text: string, prompt?: string): Promise<MultimodalResponse> {
    const response = await this.claude.messages.create({
      model: 'claude-sonnet-4-20250514',
      maxTokens: 2048,
      messages: [
        { role: 'user', content: prompt ? `${prompt}\n\n${text}` : text },
      ],
    });

    const analysis = response.content
      .filter(b => b.type === 'text')
      .map(b => (b as any).text)
      .join('');

    return { analysis };
  }

  private async processImage(
    imageBuffer: Buffer,
    prompt?: string
  ): Promise<MultimodalResponse> {
    const base64Data = imageBuffer.toString('base64');

    const response = await this.claude.messages.create({
      model: 'claude-sonnet-4-20250514',
      maxTokens: 4096,
      messages: [
        {
          role: 'user',
          content: [
            {
              type: 'image',
              source: {
                type: 'base64',
                media_type: 'image/jpeg',
                data: base64Data,
              },
            },
            { type: 'text', text: prompt || '请分析这张图片' },
          ],
        },
      ],
    });

    const analysis = response.content
      .filter(b => b.type === 'text')
      .map(b => (b as any).text)
      .join('');

    return { analysis };
  }

  private async processAudio(
    audioBuffer: Buffer,
    prompt?: string
  ): Promise<MultimodalResponse> {
    // 先转文字
    const transcript = await this.transcribeAudio(audioBuffer);

    // 再分析
    const response = await this.claude.messages.create({
      model: 'claude-sonnet-4-20250514',
      maxTokens: 2048,
      messages: [
        {
          role: 'user',
          content: `转录内容：\n${transcript}\n\n${prompt || '请分析这段音频内容'}`,
        },
      ],
    });

    const analysis = response.content
      .filter(b => b.type === 'text')
      .map(b => (b as any).text)
      .join('');

    return {
      analysis,
      structuredData: { transcript },
    };
  }

  private async processVideo(
    videoPath: string,
    prompt?: string
  ): Promise<MultimodalResponse> {
    // 提取关键帧
    const frameAnalyses = await this.analyzeVideoFrames(videoPath);

    // 汇总分析
    const response = await this.claude.messages.create({
      model: 'claude-sonnet-4-20250514',
      maxTokens: 2048,
      messages: [
        {
          role: 'user',
          content: `视频关键帧分析：\n${frameAnalyses.join('\n')}\n\n${prompt || '请总结视频内容'}`,
        },
      ],
    });

    const analysis = response.content
      .filter(b => b.type === 'text')
      .map(b => (b as any).text)
      .join('');

    return { analysis };
  }

  private async transcribeAudio(buffer: Buffer): Promise<string> {
    // 实际实现需要调用 STT API
    return '[音频转录内容]';
  }

  private async analyzeVideoFrames(path: string): Promise<string[]> {
    // 实际实现见前面章节
    return ['[帧分析结果]'];
  }
}

// 使用示例
async function demo() {
  const agent = new MultimodalAgent();

  // 处理文本
  const textResult = await agent.process({
    type: 'text',
    content: '介绍一下 TypeScript 的类型系统',
  });
  console.log('文本分析:', textResult.analysis);

  // 处理图片
  const imageBuffer = require('fs').readFileSync('./photo.jpg');
  const imageResult = await agent.process({
    type: 'image',
    content: imageBuffer,
    prompt: '描述这张照片的构图和色彩',
  });
  console.log('图像分析:', imageResult.analysis);
}
```

### Q.7.2 多模态 Tool Use

将多模态能力封装为 Tool，供 Claude 按需调用。

```typescript
const multimodalTools = [
  {
    name: 'analyze_image',
    description: '分析图片内容，提取描述、标签、文字等信息',
    input_schema: {
      type: 'object',
      properties: {
        image_path: { type: 'string', description: '图片文件路径' },
        analysis_type: {
          type: 'string',
          enum: ['description', 'ocr', 'tags', 'ui_review'],
          description: '分析类型',
        },
      },
      required: ['image_path'],
    },
  },
  {
    name: 'transcribe_audio',
    description: '将音频文件转录为文字',
    input_schema: {
      type: 'object',
      properties: {
        audio_path: { type: 'string', description: '音频文件路径' },
      },
      required: ['audio_path'],
    },
  },
  {
    name: 'extract_video_frames',
    description: '提取视频关键帧并分析',
    input_schema: {
      type: 'object',
      properties: {
        video_path: { type: 'string', description: '视频文件路径' },
        fps: { type: 'number', description: '采样帧率' },
      },
      required: ['video_path'],
    },
  },
];

async function multimodalAgentChat(userMessage: string) {
  const response = await claude.messages.create({
    model: 'claude-sonnet-4-20250514',
    maxTokens: 4096,
    tools: multimodalTools,
    messages: [{ role: 'user', content: userMessage }],
  });

  // 处理工具调用
  const toolCalls = response.content.filter(b => b.type === 'tool_use');

  if (toolCalls.length > 0) {
    const toolResults = await Promise.all(
      toolCalls.map(async (call: any) => {
        let result: string;

        switch (call.name) {
          case 'analyze_image':
            result = await executeImageAnalysis(call.input);
            break;
          case 'transcribe_audio':
            result = await executeAudioTranscription(call.input);
            break;
          case 'extract_video_frames':
            result = await executeVideoFrameExtraction(call.input);
            break;
          default:
            result = `未知工具: ${call.name}`;
        }

        return {
          type: 'tool_result',
          tool_use_id: call.id,
          content: result,
        };
      })
    );

    // 继续对话
    const finalResponse = await claude.messages.create({
      model: 'claude-sonnet-4-20250514',
      maxTokens: 4096,
      tools: multimodalTools,
      messages: [
        { role: 'user', content: userMessage },
        { role: 'assistant', content: response.content },
        { role: 'user', content: toolResults },
      ],
    });

    return finalResponse.content
      .filter(b => b.type === 'text')
      .map(b => (b as any).text)
      .join('');
  }

  return response.content
    .filter(b => b.type === 'text')
    .map(b => (b as any).text)
    .join('');
}
```

---

## Q.8 最佳实践与性能优化

### Q.8.1 Token 成本控制

多模态请求的 Token 消耗比纯文本高得多。以下是控制成本的策略：

```typescript
// 策略1: 图片压缩
async function optimizeImageForAnalysis(imagePath: string): Promise<string> {
  const { data, mediaType } = await processImage(imagePath, {
    maxWidth: 800,    // 限制尺寸
    quality: 75,      // 降低质量
    format: 'jpeg',   // 统一格式
  });
  return data;
}

// 策略2: 采样而非全量分析
async function smartBatchAnalysis(
  images: string[],
  sampleRate: number = 0.2
): Promise<string[]> {
  const sampleSize = Math.ceil(images.length * sampleRate);
  const sampled = images.sort(() => Math.random() - 0.5).slice(0, sampleSize);

  const results = await Promise.all(
    sampled.map(img => analyzeImage(img))
  );

  return results;
}

// 策略3: 缓存图像分析结果
const imageCache = new Map<string, any>();

async function cachedImageAnalysis(imagePath: string): Promise<any> {
  const hash = require('crypto')
    .createHash('md5')
    .update(require('fs').readFileSync(imagePath))
    .digest('hex');

  if (imageCache.has(hash)) {
    console.log('缓存命中:', imagePath);
    return imageCache.get(hash);
  }

  const result = await analyzeImage(imagePath);
  imageCache.set(hash, result);
  return result;
}
```

### Q.8.2 并发控制

多模态 API 调用应控制并发，避免速率限制。

```typescript
import pLimit from 'p-limit';

const limit = pLimit(5); // 最多 5 个并发

async function batchProcessImages(imagePaths: string[]): Promise<any[]> {
  const tasks = imagePaths.map(path =>
    limit(() => analyzeImage(path))
  );

  return Promise.all(tasks);
}
```

### Q.8.3 错误处理与重试

```typescript
async function robustImageAnalysis(
  imagePath: string,
  maxRetries: number = 3
): Promise<any> {
  let lastError: Error | null = null;

  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await analyzeImage(imagePath);
    } catch (error: any) {
      lastError = error;
      
      if (error.status === 429) {
        // 速率限制，等待后重试
        const waitTime = error.headers?.['retry-after'] || 60;
        console.log(`速率限制，等待 ${waitTime} 秒后重试...`);
        await new Promise(resolve => setTimeout(resolve, waitTime * 1000));
      } else if (error.status === 413) {
        // 图片太大，尝试压缩
        console.log('图片过大，尝试压缩后重试...');
        await processImage(imagePath, { maxWidth: 600, quality: 60 });
      } else {
        console.log(`尝试 ${attempt}/${maxRetries} 失败:`, error.message);
      }
    }
  }

  throw lastError;
}
```

---

## Q.9 安全与隐私考虑

### Q.9.1 敏感内容过滤

```typescript
async function safeImageAnalysis(imagePath: string): Promise<any> {
  // 先进行安全检查
  const { data, mediaType } = await processImage(imagePath, { maxWidth: 400 });

  const safetyCheck = await claude.messages.create({
    model: 'claude-sonnet-4-20250514',
    maxTokens: 256,
    messages: [
      {
        role: 'user',
        content: [
          { type: 'image', source: { type: 'base64', media_type: mediaType as any, data } },
          { type: 'text', text: '这张图片是否包含敏感内容（如个人身份信息、不当内容）？回答 YES 或 NO。' },
        ],
      },
    ],
  });

  const safetyResult = safetyCheck.content
    .filter(b => b.type === 'text')
    .map(b => (b as any).text)
    .join('')
    .toUpperCase();

  if (safetyResult.includes('YES')) {
    throw new Error('检测到敏感内容，拒绝处理');
  }

  // 安全，继续分析
  return analyzeImage(imagePath);
}
```

### Q.9.2 数据保留策略

```typescript
// 处理后删除临时文件
async function processWithCleanup<T>(
  filePath: string,
  processor: (path: string) => Promise<T>
): Promise<T> {
  try {
    const result = await processor(filePath);
    return result;
  } finally {
    // 确保删除临时文件
    if (require('fs').existsSync(filePath)) {
      require('fs').unlinkSync(filePath);
    }
  }
}
```

---

## Q.10 小结

本章系统讲解了 Claude Code SDK 的多模态应用实战：

1. **图像处理** - Base64 和 URL 两种输入方式，预处理压缩，UI 评审、OCR 提取等实战案例
2. **多图处理** - 多图比较、A/B 测试分析
3. **语音处理** - 结合 STT/TTS API 实现语音对话、会议摘要
4. **视频处理** - 帧提取、关键帧分析、内容审核
5. **多模态 Agent** - 统一处理器架构、Tool Use 集成
6. **最佳实践** - Token 控制、并发限制、错误重试
7. **安全隐私** - 敏感内容过滤、数据保留策略

多模态能力让 Claude 从"文本专家"进化为"全能助手"。结合图像理解、语音处理，你可以构建更智能、更自然的应用。

---

## 练习题

1. **图片批量标注工具**：实现一个 CLI 工具，接受目录路径，为所有图片生成描述性标签并保存为 JSON 文件。

2. **语音备忘录助手**：用户发送语音备忘录，系统自动转录、提取关键信息并生成待办事项列表。

3. **挑战题**：实现一个"智能 PPT 审查系统"——上传 PPT 文件，自动提取每页图片，分析设计一致性、内容质量，生成改进建议报告。

---

> **老三的Tips：**
> - 多模态成本更高，善用压缩和采样
> - 图像预处理是必备技能，sharp 库很好用
> - 音频/视频需要第三方 API 配合，OpenAI Whisper 是好选择
> - 缓存和并发控制是生产环境的生命线
> - 敏感内容检查不能忘，合规优先

---

_本章完，全书画上句号！🎉_
