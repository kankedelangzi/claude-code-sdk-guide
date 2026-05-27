# 附录E：版本升级指南

> 版本升级不是可选项，是必修课。SDK 版本落后，等于把 Bug 和安全漏洞带进生产环境。

## E.1 为什么要关注版本升级

Claude Code SDK 是一个**活跃开发的项目**，平均每周都有新版本发布。版本升级带来的价值：

| 升级类型 | 收益 | 风险 |
|---------|------|------|
| **补丁版本** (2.1.147 → 2.1.148) | Bug 修复、稳定性提升 | 极低 |
| **次版本** (2.1.x → 2.2.0) | 新功能、性能优化 | 中等 |
| **主版本** (2.x → 3.0) | 架构改进、Breaking Changes | 高 |

**真实案例（来自官方 Changelog）：**

```
2.1.148 (2026-05-22)
  * 修复了 Bash 工具在某些用户环境下返回 exit code 127 的问题
  → 如果你遇到 "command not found" 错误，升级就对了

2.1.147 (2026-05-21)
  * 改进了自动更新器：重试 transient 网络故障
  * 修复了 PowerShell 工具在 Windows 上失败的问题
  → Windows 用户必升

2.1.145 (2026-05-19)
  * 修复了 Bash 命令中裸变量赋值被自动批准的安全漏洞
  → 安全修复，强烈建议升级
```

---

## E.2 检查当前版本

### E.2.1 CLI 版本

```bash
# 查看 CLI 版本
claude --version

# 输出示例：
# 2.1.148
```

### E.2.2 SDK 版本

```bash
# npm 项目
npm list @anthropic-ai/claude-code

# yarn 项目
yarn list --pattern @anthropic-ai/claude-code

# pnpm 项目
pnpm list @anthropic-ai/claude-code

# 输出示例：
# @anthropic-ai/claude-code@2.1.147
```

### E.2.3 程序中检测版本

```typescript
// version-check.ts
import { version } from '@anthropic-ai/claude-code/package.json';

console.log(`当前 SDK 版本：${version}`);

// 版本比较
const [major, minor, patch] = version.split('.').map(Number);

if (major < 2 || (major === 2 && minor < 1)) {
  console.warn('⚠️  版本过旧，建议升级到 2.1+');
}
```

---

## E.3 理解版本号（Semantic Versioning）

Claude Code SDK 遵循 **SemVer** 规范：

```
2  .  1  .  148
│     │     │
│     │     └─ 补丁版本 (patch)：Bug 修复，向后兼容
│     └─────── 次版本 (minor)：新功能，向后兼容
└───────────── 主版本 (major)：破坏性变更
```

### 版本号规则

| 变更类型 | 版本号变化 | 你的代码需要改吗？ |
|---------|-----------|------------------|
| Bug 修复 | 2.1.147 → 2.1.148 | ❌ 不需要 |
| 新功能 | 2.1.x → 2.2.0 | ⚠️ 可能需要（如果用了新 API） |
| Breaking Change | 2.x → 3.0 | ✅ 必须改 |

**经验法则：**
- `npm update` 安全（只升级补丁和次版本）
- `npm install @anthropic-ai/claude-code@latest` 危险（可能升主版本）

---

## E.4 升级准备清单

**在升级之前，完成这个检查清单：**

```markdown
## 升级前检查清单

- [ ] 备份当前代码（git commit 或分支）
- [ ] 检查 CHANGELOG.md 中的 Breaking Changes
- [ ] 在测试环境先升级验证
- [ ] 确认依赖的其他包兼容新版本
- [ ] 准备回滚方案（package-lock.json 备份）
- [ ] 通知团队成员即将升级
- [ ] 检查 Node.js 版本要求是否变化
```

### 实操步骤

```bash
# Step 1: 备份当前依赖版本
cp package-lock.json package-lock.json.backup

# Step 2: 创建升级分支
git checkout -b chore/upgrade-claude-code-$(date +%Y%m%d)

# Step 3: 查看 CHANGELOG
open https://code.claude.com/docs/en/changelog

# Step 4: 在测试环境升级
npm install @anthropic-ai/claude-code@latest --save-dev

# Step 5: 运行测试
npm test

# Step 6: 如果测试通过，提交代码
git add package.json package-lock.json
git commit -m "chore: upgrade @anthropic-ai/claude-code to latest"
```

---

## E.5 常见破坏性变更（Breaking Changes）

虽然 Claude Code SDK 目前还在 2.x 阶段，但了解常见的破坏性变更模式能帮你快速适配。

### E.5.1 API 签名变化

**假设场景：** `Client` 构造函数参数变化

```typescript
// ❌ 旧版本代码（2.0）
const client = new Client({
  apiKey: 'sk-ant-xxx',
  model: 'claude-opus-4-20241120'
});

// ✅ 新版本代码（2.1+）
const client = new Client({
  apiKey: 'sk-ant-xxx',
  defaultModel: 'claude-opus-4-20241120'  // model → defaultModel
});
```

**如何发现：**
1. 运行代码，看 TypeScript 编译错误
2. 查看 CHANGELOG 中的 "Breaking Changes" 部分
3. 运行 `tsc --noEmit` 检查类型错误

### E.5.2 工具名称变化

**真实案例（来自 2.1.147）：**

```typescript
// ❌ 旧版本（2.1.146 及之前）
// /simplify 命令用于代码审查和清理

// ✅ 新版本（2.1.147+）
// /simplify 重命名为 /code-review
// 旧命令移除，必须使用新命令
```

**适配代码：**

```typescript
// 如果你的代码中硬编码了 /simplify
// 需要全局替换成 /code-review
```

### E.5.3 配置项废弃

```typescript
// ❌ 废弃的配置（2.0）
const client = new Client({
  enableBetaFeatures: true  // 已废弃
});

// ✅ 新配置（2.1+）
const client = new Client({
  experimental: {
    enableBetaFeatures: true
  }
});
```

---

## E.6 测试策略

升级后必须跑测试，但这里有一套**分层测试策略**，帮你快速定位问题。

### E.6.1 单元测试

```typescript
// __tests__/sdk-upgrade.test.ts
import { Client } from '@anthropic-ai/claude-code';

describe('SDK 升级验证', () => {
  test('基本对话功能', async () => {
    const client = new Client();
    const response = await client.messages.create({
      model: 'claude-opus-4-20241120',
      max_tokens: 1024,
      messages: [{ role: 'user', content: 'Hello' }]
    });
    
    expect(response).toBeDefined();
    expect(response.content).toBeTruthy();
  });

  test('工具调用功能', async () => {
    const client = new Client();
    const response = await client.messages.create({
      model: 'claude-opus-4-20241120',
      max_tokens: 1024,
      messages: [{ role: 'user', content: '当前时间' }],
      tools: [
        {
          name: 'get_time',
          description: '获取当前时间',
          input_schema: { type: 'object', properties: {} }
        }
      ]
    });
    
    expect(response.stop_reason).toBe('tool_use');
  });
});
```

运行测试：

```bash
npm test -- --testPathPattern=sdk-upgrade
```

### E.6.2 集成测试

```typescript
// __tests__/integration/code-review.test.ts
import { execSync } from 'child_process';

describe('代码审查集成测试', () => {
  test('CLI 命令可用', () => {
    const output = execSync('claude --version').toString();
    expect(output).toMatch(/\d+\.\d+\.\d+/);
  });

  test('/code-review 命令可用', async () => {
    // 模拟运行 /code-review 命令
    const output = execSync('claude /code-review --help').toString();
    expect(output).toContain('code-review');
  });
});
```

### E.6.3 手动验证清单

```markdown
## 升级后手动验证清单

- [ ] 启动开发服务器，无错误
- [ ] 发送一条测试消息，收到回复
- [ ] 测试文件上传功能
- [ ] 测试工具调用（如果有）
- [ ] 检查日志，无异常
- [ ] 运行完整 E2E 测试套件
```

---

## E.7 回滚方案

如果升级后出现问题，你需要快速回滚。

### E.7.1 使用 package-lock.json 回滚

```bash
# 假设你备份了 package-lock.json
cp package-lock.json.backup package-lock.json

# 重新安装依赖
npm ci

# 验证版本
npm list @anthropic-ai/claude-code
```

### E.7.2 指定版本重新安装

```bash
# 回滚到特定版本
npm install @anthropic-ai/claude-code@2.1.145 --save-dev

# 提交回滚
git add package.json package-lock.json
git commit -m "revert: rollback @anthropic-ai/claude-code to 2.1.145"
```

### E.7.3 Git 回滚

```bash
# 如果刚提交升级代码，直接 revert
git revert HEAD

# 或者丢弃升级提交
git reset --hard HEAD~1
```

---

## E.8 真实升级案例

### 案例1：从 2.1.145 升级到 2.1.148

**背景：** 你在使用 2.1.145，发现 Windows 上 PowerShell 工具失败。

**CHANGELOG 分析：**

```
2.1.148 (2026-05-22)
  * 修复了 Bash 工具返回 exit code 127 的问题

2.1.147 (2026-05-21)
  * 修复了 PowerShell 工具在 Windows 上失败的问题 ✅ 你需要这个
  * 改进了自动更新器

2.1.146 (2026-05-20)
  * 无相关修复
```

**结论：** 升级到 2.1.147+ 即可。

**操作步骤：**

```bash
# 1. 检查当前版本
claude --version  # 输出 2.1.145

# 2. 升级
npm install @anthropic-ai/claude-code@2.1.148 --save-dev

# 3. 验证
claude --version  # 输出 2.1.148

# 4. 测试 PowerShell 工具
claude
> /powershell Get-Process

# 5. 如果正常工作，提交
git add package.json package-lock.json
git commit -m "chore: upgrade to 2.1.148 (fix PowerShell issue)"
```

---

### 案例2：安全漏洞修复升级

**背景：** CHANGELOG 中提到安全修复。

**CHANGELOG 分析：**

```
2.1.145 (2026-05-19)
  * 修复了 Bash 命令中裸变量赋值被自动批准的安全漏洞 ⚠️ 安全修复
```

**结论：** 这是**强制升级**，即使测试不充分也要升。

**操作步骤：**

```bash
# 1. 立即升级（安全优先）
npm install @anthropic-ai/claude-code@latest --save-dev

# 2. 快速验证核心功能
npm test -- --testPathPattern=core

# 3. 提交（即使测试不完整）
git add package.json package-lock.json
git commit -m "security: upgrade to latest (security fix)"

# 4. 推送后立即部署
git push && npm run deploy
```

---

## E.9 长期维护策略

### E.9.1 自动化版本检查

```json
// package.json
{
  "scripts": {
    "upgrade:check": "npm outdated @anthropic-ai/claude-code",
    "upgrade:safe": "npm update @anthropic-ai/claude-code",
    "upgrade:latest": "npm install @anthropic-ai/claude-code@latest --save-dev"
  }
}
```

定期运行：

```bash
# 每周检查一次
npm run upgrade:check

# 输出示例：
# Package                     Current   Wanted   Latest
# @anthropic-ai/claude-code   2.1.145  2.1.148  2.1.148
```

### E.9.2 使用 Renovate / Dependabot

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    allow:
      - dependency-name: "@anthropic-ai/claude-code"
    ignore:
      - dependency-name: "@anthropic-ai/claude-code"
        versions: ["3.x"]  # 忽略主版本升级
```

### E.9.3 锁定版本范围

```json
// package.json
{
  "devDependencies": {
    // ✅ 推荐：允许补丁和次版本升级
    "@anthropic-ai/claude-code": "^2.1.145",
    
    // ⚠️ 谨慎：锁定 exact 版本
    // "@anthropic-ai/claude-code": "2.1.145",
    
    // ❌ 避免：使用 latest
    // "@anthropic-ai/claude-code": "latest"
  }
}
```

**版本范围说明：**

```
^2.1.145  → 允许 2.1.145 ~ <3.0.0
~2.1.145  → 允许 2.1.145 ~ <2.2.0
2.1.145   → 锁定 exact 版本
latest    → 始终最新（危险！）
```

---

## E.10 小结

版本升级的最优策略：

1. **补丁版本**：立即升级（低风险，有 Bug 修复）
2. **次版本**：测试后升级（有新功能，可能需适配）
3. **主版本**：谨慎升级（有 Breaking Changes）

**升级三部曲：**
```
1. 检查 CHANGELOG → 了解变更内容
2. 测试环境验证 → 确保无回归
3. 生产环境部署 → 监控异常情况
```

**自动化工具：**
- `npm outdated` 检查更新
- `dependabot` 自动提 PR
- `npm audit` 检查安全漏洞

> 版本升级不是"是否"的问题，是"何时"的问题。主动升级，好过被动救火。

---

**下一步：** 附录F 将介绍 SDK 的性能基准测试数据，帮你选择最合适的模型和配置。
