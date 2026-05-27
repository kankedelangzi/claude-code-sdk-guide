# 第16章：插件开发指南

> 把你的创意打包成插件，让整个团队都能用上。

## 16.1 什么是 Claude Code 插件

Claude Code 插件（Plugin）是一个**自包含的目录**，里面可以包含 Skills、Agents、Hooks、MCP Servers、LSP Servers 等组件。核心目的是**封装和分发**——把单个项目的配置变成团队共享的可复用包。

### 插件 vs 独立配置

| 方式 | 技能命名 | 适用场景 |
|------|---------|---------|
| **独立配置**（`.claude/` 目录） | `/hello` | 个人项目、快速实验 |
| **插件**（`.claude-plugin/plugin.json`） | `/my-plugin:hello` | 团队共享、版本发布、多项目复用 |

### 插件组件一览

| 目录/文件 | 用途 |
|-----------|------|
| `.claude-plugin/plugin.json` | 插件清单（名称、描述、版本） |
| `skills/` | 技能，`<name>/SKILL.md` 形式 |
| `agents/` | 自定义 Agent 定义 |
| `hooks/hooks.json` | 事件钩子处理器 |
| `.mcp.json` | MCP Server 配置 |
| `.lsp.json` | LSP 语言服务器配置 |
| `monitors/monitors.json` | 后台监控器 |
| `bin/` | 可执行文件，自动加入 PATH |
| `settings.json` | 插件启用时的默认设置 |

> ⚠️ **常见错误**：不要把 `skills/`、`agents/`、`hooks/` 放进 `.claude-plugin/` 里。只有 `plugin.json` 在其中，其余组件都在插件根目录。

---

## 16.2 创建你的第一个插件

### 16.2.1 创建目录和清单

```bash
mkdir -p my-team-plugin/.claude-plugin
mkdir -p my-team-plugin/skills/hello
```

```json
// my-team-plugin/.claude-plugin/plugin.json
{
  "name": "my-team-plugin",
  "description": "团队开发规范插件：代码审查、格式化、部署",
  "version": "1.0.0",
  "author": { "name": "老三" }
}
```

- **name**：唯一标识符 + 技能命名空间前缀
- **version**：只有 bump 版本号时用户才会收到更新

### 16.2.2 添加 Skill

```bash
cat > my-team-plugin/skills/hello/SKILL.md << 'EOF'
---
description: 向用户打招呼并介绍当前项目状态
disable-model-invocation: true
---

向用户热情地打招呼，简要说明当前 Git 仓库的分支和最近一次提交。
运行 `git branch --show-current` 和 `git log -1 --oneline`，然后用友好语气汇报。
EOF
```

`disable-model-invocation: true` 表示只能手动调用，Claude 不会自动触发。

### 16.2.3 本地测试

```bash
claude --plugin-dir ./my-team-plugin
# 输入 /my-team-plugin:hello 测试
# 修改后用 /reload-plugins 热加载
```

---

## 16.3 实战：代码审查插件

一个完整的插件，包含 Skill + Agent + Hook + bin 四个组件。

### 16.3.1 插件结构

```
code-review-plugin/
├── .claude-plugin/plugin.json
├── skills/review/SKILL.md
├── agents/security-reviewer.md
├── hooks/hooks.json
└── bin/check-style.sh
```

### 16.3.2 plugin.json

```json
{
  "name": "code-review-plugin",
  "description": "代码审查插件：自动检查代码风格、安全问题和最佳实践",
  "version": "1.2.0",
  "author": { "name": "老三" },
  "repository": "https://github.com/example/code-review-plugin"
}
```

### 16.3.3 审查 Skill

```markdown
// skills/review/SKILL.md
---
description: 审查代码变更，检查风格、安全性和最佳实践。当用户要求代码审查或 PR Review 时使用。
---

# 代码审查 Skill

审查维度：
1. 代码风格：命名、缩进、注释
2. 错误处理：未处理异常、空 catch 块
3. 安全问题：硬编码密钥、SQL 注入、XSS
4. 性能：N+1 查询、不必要循环
5. 测试覆盖

输出分级：
- 🔴 严重：必须修复
- 🟡 建议：推荐修复
- 🟢 可选：锦上添花

流程：指定文件则审查该文件，否则 `git diff HEAD~1` 检查最近变更。
```

### 16.3.4 安全审查 Agent

```markdown
// agents/security-reviewer.md
---
name: security-reviewer
description: 专注安全审查，涉及安全代码时自动触发
model: sonnet
effort: high
maxTurns: 15
disallowedTools: Write, Edit
---

你是安全审查专家，检查 OWASP Top 10 漏洞、输入校验、认证授权、敏感数据处理。
只能读取代码不能修改。审查完成后给出安全评分（A-F）和详细报告。
```

> `disallowedTools: Write, Edit` 确保审查者只读不改——审查和修改分离是好的安全实践。

### 16.3.5 自动格式化 Hook

```json
// hooks/hooks.json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '.tool_input.file_path | select(endswith(\".ts\") or endswith(\".js\"))' | xargs -r npx prettier --write 2>/dev/null || true"
          }
        ]
      }
    ]
  }
}
```

每次 Claude 编辑 JS/TS 文件后自动格式化。

### 16.3.6 风格检查脚本

```bash
#!/bin/bash
# bin/check-style.sh — 插件启用时自动加入 PATH
TARGET="${1:-.}"
echo "=== ESLint ==="
npx eslint "$TARGET" --format stylish 2>/dev/null || echo "ESLint 未配置"
echo "=== Prettier ==="
npx prettier --check "$TARGET" 2>/dev/null || echo "发现格式问题"
echo "=== TypeScript ==="
npx tsc --noEmit 2>/dev/null || echo "检查完成"
```

```bash
chmod +x bin/check-style.sh
```

> `bin/` 下的可执行文件自动加入 Bash 的 PATH，Skill 和 Agent 可以直接调用。

---

## 16.4 Hook 深度指南

Hook 是插件的"神经系统"——在 Claude Code 生命周期关键节点自动执行代码。

### 16.4.1 常用事件

| 事件 | 触发时机 | 典型用途 |
|------|---------|---------|
| `SessionStart` | 会话开始/恢复 | 注入上下文 |
| `PreToolUse` | 工具调用之前 | 阻止危险操作 |
| `PostToolUse` | 工具调用之后 | 自动格式化 |
| `Notification` | Claude 需要用户注意 | 桌面通知 |
| `Stop` | Claude 完成回复 | 自动保存、触发 CI |
| `FileChanged` | 文件磁盘变化 | 热重载 |

### 16.4.2 Hook 的三种类型

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command", "command": "my-check.sh" },
          { "type": "http", "url": "https://api.example.com/check" },
          { "type": "prompt", "prompt": "检查命令安全性: $ARGUMENTS" }
        ]
      }
    ]
  }
}
```

- **command**：执行 shell 命令，最常用
- **http**：POST 事件 JSON 到 URL，对接外部服务
- **prompt**：用 LLM 判断，适合需要"理解"的场景

### 16.4.3 阻止危险命令

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "command",
        "if": "Bash(rm *)",
        "command": "${CLAUDE_PROJECT_DIR}/hooks/block-rm.sh"
      }]
    }]
  }
}
```

```bash
#!/bin/bash
# block-rm.sh
COMMAND=$(jq -r '.tool_input.command')
if echo "$COMMAND" | grep -q 'rm -rf'; then
  jq -n '{
    hookSpecificOutput: {
      hookEventName: "PreToolUse",
      permissionDecision: "deny",
      permissionDecisionReason: "危险命令被安全策略阻止"
    }
  }'
else
  exit 0
fi
```

退出码：0 = 无决策；2 = 阻止操作。

---

## 16.5 MCP Server 集成

插件可以打包 MCP Server，让 Claude 自动获得与外部服务交互的能力。

```json
// .mcp.json
{
  "mcpServers": {
    "plugin-database": {
      "command": "${CLAUDE_PLUGIN_ROOT}/servers/db-server",
      "args": ["--config", "${CLAUDE_PLUGIN_ROOT}/config.json"],
      "env": { "DB_PATH": "${CLAUDE_PLUGIN_ROOT}/data" }
    }
  }
}
```

`${CLAUDE_PLUGIN_ROOT}` 是 Claude Code 提供的环境变量，指向插件根目录，确保路径在任何项目下都正确解析。

插件 MCP Server 在插件启用时自动启动，与用户配置的 MCP Server 互不干扰。

---

## 16.6 后台监控器

监控器让插件在后台持续关注日志、文件或外部状态，有事件时通知 Claude。

```json
// monitors/monitors.json
[
  {
    "name": "error-log",
    "command": "tail -F ./logs/error.log",
    "description": "应用错误日志监控"
  },
  {
    "name": "test-watcher",
    "command": "fswatch -o ./src | xargs -n1 npm test",
    "description": "文件变更自动跑测试"
  }
]
```

Claude Code 会话期间自动启动每个 monitor，stdout 的每一行作为通知推送给 Claude。

---

## 16.7 插件默认设置

插件可以包含 `settings.json`，启用时自动应用配置：

```json
// settings.json
{
  "agent": "security-reviewer"
}
```

这会让 `security-reviewer` Agent 成为默认主线程，改变 Claude Code 的默认行为。

---

## 16.8 分发与安装

### 通过 Git URL 安装

```bash
claude plugin add https://github.com/example/code-review-plugin
```

### 通过插件市场

在 Claude Code 中运行 `/plugin`，切换到 Discover 标签页搜索安装。

### 版本管理

- 设置了 `version` 字段：只有 bump 版本号用户才收到更新
- 省略 `version`：每个 git commit 都算新版本

---

## 16.9 最佳实践

1. **命名空间避免冲突**：插件名要有辨识度，避免和其他插件 Skill 重名
2. **Agent 限制权限**：安全审查 Agent 不需要写权限就用 `disallowedTools`
3. **Hook 用 `if` 条件过滤**：减少不必要的进程启动
4. **bin/ 放工具脚本**：自动加入 PATH，Skill 和 Agent 都能调用
5. **`${CLAUDE_PLUGIN_ROOT}` 解析路径**：不要硬编码绝对路径
6. **先 standalone 再插件化**：在 `.claude/` 验证可行后再打包成插件
7. **版本号管理**：发布前 bump `version`，避免用户频繁收到无意义更新

---

## 16.10 小结

本章覆盖了 Claude Code 插件开发的完整链路：从创建基础插件、编写 Skill 和 Agent，到配置 Hook、集成 MCP Server、添加后台监控器，最后分发安装。插件的核心价值在于**把个人经验变成团队资产**——你在某个项目中积累的最好实践，通过插件可以零成本复制到所有项目。

> 下一章我们将深入 **SDK 的子进程管理与并发控制**，这是构建复杂 Claude Code 应用的关键技术。