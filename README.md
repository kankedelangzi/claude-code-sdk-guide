# 📚 Claude Code SDK 编程指南

> 从零到精通，带你玩转 Claude Code SDK！

![License](https://img.shields.io/badge/license-MIT-blue.svg) ![Status](https://img.shields.io/badge/status-写作中-orange.svg) ![Chapters](https://img.shields.io/badge/章节-23%2F30-green.svg)

---

## 🎯 关于这本书

《Claude Code SDK 编程指南》是一本面向开发者的实战手册，涵盖从入门到企业级应用的全链路知识。

### 核心特色
- ✅ **可运行代码** — 每章都有完整可执行的示例
- ✅ **深度优先** — 不泛泛而谈，讲透一个主题
- ✅ **实战导向** — 真实场景、真实问题、真实方案
- ✅ **持续更新** — 每10分钟写一节，AI 辅助创作

---

## 📖 目录（已完成 23/30 章）

### 基础篇
1. [引言：为什么选择 Claude Code SDK](book/01-introduction.md)
2. [环境搭建与安装](book/02-installation.md)
3. [第一个程序：Hello Claude](book/03-first-program.md)
4. [核心概念与架构](book/04-concepts.md)

### 进阶篇
5. [消息与内容类型](book/05-message-content.md)
6. [工具定义与调用](book/06-tool-definition.md)
7. [MCP 协议详解](book/07-mcp-protocol.md)
8. [CLAUDE.md 配置文件](book/08-claudemd-config.md)
9. [多轮对话管理](book/09-multi-turn-conversation.md)
10. [智能助手实战](book/10-smart-assistant.md)
11. [代码生成器开发](book/11-code-generator.md)
12. [性能优化技巧](book/12-performance-best-practices.md)
13. [高级提示词工程](book/13-advanced-prompt-engineering.md)

### 高级篇
14. [多智能体协作](book/14-multi-agent-collaboration.md)
15. [生产环境部署](book/15-production-deployment.md)
16. [插件开发指南](book/16-plugin-development.md)
17. [调试与故障排查](book/17-debugging-troubleshooting.md)
18. [流式输出与 SSE](book/18-streaming-sse.md)
19. [成本优化策略](book/19-cost-optimization.md)
20. [测试策略与 Mock](book/20-testing-strategy-mock.md)
21. [错误处理与重试](book/21-error-handling-retry.md)
22. [缓存策略与性能](book/22-cache-strategy-performance.md)
23. [企业级应用场景](book/23-enterprise-use-cases.md)

### 附录
- [附录A：API 参考](book/appendix-a-api-reference.md)
- [附录B：常见问题](book/appendix-b-faq.md)
- [附录C：CLI vs SDK](book/appendix-c-cli-vs-sdk.md)
- [附录D：代码示例集](book/appendix-d-code-examples.md)
- [附录E：版本升级指南](book/appendix-e-version-upgrade.md)
- [附录F：安全最佳实践](book/appendix-f-security-best-practices.md)
- [附录G：性能基准测试](book/appendix-g-performance-benchmark.md)

👉 **[查看完整目录](book/SUMMARY.md)** | **[待写主题](book/TODO.md)**

---

## 🚀 快速开始

### 环境要求
- Node.js >= 18
- npm 或 yarn
- Anthropic API Key

### 安装 SDK
\`\`\`bash
npm install @anthropic-ai/sdk
\`\`\`

### 第一个示例
\`\`\`javascript
import Anthropic from '@anthropic-ai/sdk';

const client = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY,
});

const message = await client.messages.create({
  model: 'claude-3-5-sonnet-20241022',
  max_tokens: 1024,
  messages: [{ role: 'user', content: 'Hello Claude!' }],
});

console.log(message.content[0].text);
\`\`\`

👉 更多示例见 [第3章](book/03-first-program.md)

---

## 📊 项目进度

| 指标 | 数值 |
|------|------|
| 已完成章节 | 23 / 30 |
| 总字数 | ~45万字 |
| 代码示例 | 150+ |
| 完成度 | 77% |

---

## 🤝 贡献指南

本书采用 **AI 辅助写作**，由 [老三](https://github.com/kankedelangzi)（Claude Code SDK 专家）主笔。

### 如何贡献
1. Fork 本仓库
2. 创建你的分支 (`git checkout -b feature/AmazingFeature`)
3. 提交你的修改 (`git commit -m 'Add some AmazingFeature'`)
4. 推送到分支 (`git push origin feature/AmazingFeature`)
5. 打开一个 Pull Request

### 问题反馈
- 发现有错误？[提 Issue](https://github.com/kankedelangzi/ai-agent/issues)
- 想补充内容？欢迎 PR！

---

## 📄 许可证

本项目采用 MIT 许可证 - 详见 [LICENSE](LICENSE) 文件。

---

## 🙏 致谢

- **Anthropic** — 提供强大的 Claude API
- **OpenClaw 团队** — 提供 AI Agent 基础设施
- **所有贡献者** — 让这本书越来越好

---

## 📮 联系方式

- GitHub: [@kankedelangzi](https://github.com/kankedelangzi)
- 邮箱: （待补充）

---

**⚡ Powered by [OpenClaw](https://github.com/openclaw/openclaw) + Claude Sonnet 4.5**
