# 第7章：MCP 协议详解

> "一个人再能干，也需要工具箱。MCP 就是给 Claude 发的那把万能钥匙。" —— 老三

前面几章，我们学会了定义 Tool 让 Claude 调用本地函数。但真实世界里，你的助手需要连接外部系统——数据库、API、第三方服务。一个一个手写 Tool 太累，而且没法复用。

**MCP（Model Context Protocol）** 就是解决这个问题的开放协议。它让 Claude Code 能连接任意 MCP Server，瞬间获得几十甚至上百个新能力——查 JIRA、读 Notion、操作 GitHub、查询数据库……插上就能用，不用写一行代码。

---

## 7.1 MCP 协议概述

### 7.1.1 什么是 MCP

MCP 全称 **Model Context Protocol**，是 Anthropic 主导的开源协议，专为 AI 与外部工具/数据源的集成而设计。

类比一下：如果 Claude 是一部手机，MCP 就是 USB-C 接口——不管什么外设（键盘、显示器、硬盘），只要遵循接口协议，插上就能用。

### 7.1.2 MCP 的三大能力

一个 MCP Server 可以提供三种能力：

| 能力 | 说明 | 类比 |
|------|------|------|
| **Resources** | 可读取的数据（文件、API 响应） | 只读数据库 |
| **Tools** | 可调用的函数（需用户批准） | 远程过程调用 |
| **Prompts** | 预写模板，帮助完成特定任务 | 快捷指令 |

其中 **Tools** 是最常用的能力，也是本章重点。

### 7.1.3 架构一览

```
┌─────────────┐     MCP 协议      ┌──────────────┐
│  Claude Code │ ◄──────────────► │  MCP Server   │
│  (客户端)    │   JSON-RPC/HTTP   │  (工具提供方)  │
└─────────────┘                   └──────┬───────┘
                                         │
                                    ┌────▼────┐
                                    │ 外部服务  │
                                    │ GitHub   │
                                    │ Notion   │
                                    │ Database │
                                    └─────────┘
```

核心要点：
- Claude Code 是 **MCP Client**
- MCP Server 是独立的进程或远程服务
- 两者通过 **JSON-RPC** 通信
- 一个 Client 可以同时连接多个 Server

---

## 7.2 连接 MCP Server

Claude Code 支持三种传输方式连接 MCP Server：

### 7.2.1 远程 HTTP Server（推荐）

适用于云端托管的 MCP 服务，最主流的方式：

```bash
# 基本语法
claude mcp add --transport http <名称> <URL>

# 实战示例：连接 Notion
claude mcp add --transport http notion https://mcp.notion.com/mcp

# 带认证头
claude mcp add --transport http secure-api https://api.example.com/mcp \
  --header "Authorization: Bearer sk-your-token-here"
```

### 7.2.2 远程 SSE Server（已弃用）

SSE 传输已被弃用，新项目请用 HTTP：

```bash
# 仅在你必须连接旧版 SSE 服务时使用
claude mcp add --transport sse legacy-api https://api.example.com/sse
```

### 7.2.3 本地 Stdio Server

本地进程模式，适合需要直接访问系统资源的工具：

```bash
# 基本语法（注意 -- 的位置！）
claude mcp add [选项] <名称> -- <命令> [参数...]

# 实战示例：添加 Airtable MCP Server
claude mcp add --transport stdio \
  --env AIRTABLE_API_KEY=your_key \
  airtable -- npx -y airtable-mcp-server

# Python 服务器示例
claude mcp add --transport stdio \
  --env DB_PATH=/data/app.db \
  my-db -- python db_server.py
```

⚠️ **选项顺序很重要！** 所有选项（`--transport`、`--env`、`--scope`）必须放在服务器名称**之前**，`--` 之后才是传给服务器的命令和参数。

### 7.2.4 管理已连接的 Server

```bash
# 列出所有已配置的 Server
claude mcp list

# 查看某个 Server 详情
claude mcp get notion

# 移除 Server
claude mcp remove notion

# 在 Claude Code 会话中检查 Server 状态
/mcp
```

`/mcp` 命令会显示每个已连接 Server 的工具数量，以及是否有异常。

---

## 7.3 用 JSON 配置文件管理 Server

除了命令行，你还可以用 JSON 配置文件管理 MCP Server。适合团队共享和版本控制。

### 7.3.1 项目级配置：`.mcp.json`

在项目根目录创建 `.mcp.json`，所有团队成员共享：

```json
{
  "mcpServers": {
    "github": {
      "type": "http",
      "url": "https://mcp.github.com/mcp"
    },
    "my-local-tool": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "my-mcp-server"],
      "env": {
        "API_KEY": "${MY_API_KEY}"
      }
    }
  }
}
```

### 7.3.2 用户级配置：`~/.claude.json`

全局生效，所有项目都可用：

```json
{
  "mcpServers": {
    "my-global-tool": {
      "type": "http",
      "url": "https://api.mytool.com/mcp",
      "headers": {
        "Authorization": "Bearer ${MY_TOKEN}"
      }
    }
  }
}
```

### 7.3.3 用 `claude mcp add-json` 添加

```bash
claude mcp add-json my-server '{
  "type": "stdio",
  "command": "python",
  "args": ["server.py"],
  "env": {
    "DB_URL": "postgres://localhost/mydb"
  }
}'
```

---

## 7.4 动态工具更新

Claude Code 支持 MCP 的 `list_changed` 通知机制。这意味着：

- MCP Server 可以在运行时动态添加或移除工具
- Claude Code 会自动感知变化，无需重启
- 你不需要提前知道 Server 提供哪些工具——连上再说

这也是 MCP 比硬编码 Tool 定义灵活得多的关键原因。

---

## 7.5 动手构建一个 MCP Server

理论讲完了，来写一个真正的 MCP Server。我们用 Python 的 `FastMCP` 框架，构建一个简单的"计算器 Server"，提供 `add` 和 `multiply` 两个工具。

### 7.5.1 项目初始化

```bash
# 创建项目
mkdir calculator-mcp && cd calculator-mcp

# 使用 uv 管理依赖
uv init
uv add "mcp[cli]"

# 创建服务器文件
touch calculator.py
```

### 7.5.2 完整服务器代码

```python
# calculator.py
"""一个简单的 MCP 计算器 Server"""

from mcp.server.fastmcp import FastMCP

# 初始化 MCP Server
mcp = FastMCP("calculator")

@mcp.tool()
def add(a: float, b: float) -> float:
    """两个数相加。

    Args:
        a: 第一个数
        b: 第二个数
    """
    return a + b

@mcp.tool()
def multiply(a: float, b: float) -> float:
    """两个数相乘。

    Args:
        a: 第一个数
        b: 第二个数
    """
    return a * b

@mcp.tool()
def power(base: float, exponent: float) -> float:
    """计算 base 的 exponent 次方。

    Args:
        base: 底数
        exponent: 指数
    """
    return base ** exponent

if __name__ == "__main__":
    # 以 stdio 模式启动（Claude Code 默认模式）
    mcp.run(transport="stdio")
```

### 7.5.3 本地测试

```bash
# 用 MCP Inspector 测试你的 Server
npx @anthropic/mcp-inspector python calculator.py
```

MCP Inspector 会打开一个 Web UI，你可以测试每个工具的调用。

### 7.5.4 接入 Claude Code

```bash
# 连接本地 Server
claude mcp add --transport stdio calculator -- python calculator.py

# 验证连接
claude mcp get calculator

# 在 Claude Code 中检查
/mcp
```

连接成功后，你就可以在 Claude Code 中这样对话：

```
你：帮我算一下 (12.5 + 3.7) * 2 的结果

Claude：我来调用 calculator 的工具计算……
[调用 add(12.5, 3.7) → 16.2]
[调用 multiply(16.2, 2) → 32.4]

结果是 32.4。
```

### 7.5.5 日志的坑

对于 stdio 模式的 Server，**千万不要往 stdout 打日志！** 因为 stdout 被用于 JSON-RPC 通信，写日志会破坏协议。

```python
import sys
import logging

# ❌ 错误！这会破坏 MCP 协议
print("Processing request")

# ✅ 正确：写到 stderr
print("Processing request", file=sys.stderr)

# ✅ 正确：用 logging（默认输出到 stderr）
logging.info("Processing request")
```

---

## 7.6 MCP Server 进阶：带认证的 HTTP Server

对于生产环境，通常需要 HTTP Server + 认证。这里用 FastMCP 的 HTTP 模式：

```python
# auth_server.py
"""带 Bearer Token 认证的 MCP HTTP Server"""

import os
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("secure-tools")

@mcp.tool()
def get_secret_info(project: str) -> str:
    """获取项目的机密信息（模拟）。

    Args:
        project: 项目名称
    """
    return f"项目 {project} 的部署密钥: sk-prod-{hash(project)}"

if __name__ == "__main__":
    mcp.run(transport="http", host="0.0.0.0", port=8000)
```

```bash
# 启动 Server
python auth_server.py

# Claude Code 客户端连接（带 Bearer Token）
claude mcp add --transport http secure-tools \
  http://localhost:8000/mcp \
  --header "Authorization: Bearer my-secret-token"
```

---

## 7.7 实战场景

连接 MCP Server 后，Claude Code 的能力会大幅扩展。以下是真实场景：

### 场景1：JIRA + GitHub 联动

```bash
# 连接 JIRA
claude mcp add --transport http jira https://mcp.atlassian.com/mcp \
  --header "Authorization: Bearer ${JIRA_TOKEN}"

# 连接 GitHub
claude mcp add --transport http github https://mcp.github.com/mcp
```

然后你可以对 Claude 说：

> "查看 JIRA 上的 ENG-4521，然后创建一个对应 feature 分支并提交 PR。"

Claude 会自动：读 JIRA → 理解需求 → 创建分支 → 写代码 → 提 PR。全程零手动操作。

### 场景2：数据库查询

```bash
claude mcp add --transport stdio \
  --env DATABASE_URL=postgres://localhost/myapp \
  mydb -- npx -y @anthropic/mcp-server-postgres
```

> "找出最近 7 天注册用户数最多的 10 个城市。"

### 场景3：监控 + 告警

```bash
claude mcp add --transport http sentry https://mcp.sentry.io/mcp \
  --header "Authorization: Bearer ${SENTRY_TOKEN}"
```

> "检查 Sentry 最近一小时的错误，如果有新的 P0 告警，帮我创建 Slack 通知。"

---

## 7.8 安全注意事项

⚠️ **连接外部 MCP Server 有安全风险：**

1. **信任审查**：只连接你信任的 Server。Server 可以读取你的文件、执行命令。
2. **Prompt 注入**：从外部获取内容可能包含恶意指令。MCP Server 返回的内容可能被注入。
3. **权限最小化**：给 MCP Server 最少的环境变量和权限。
4. **审计日志**：定期检查 `/mcp` 面板，确认 Server 状态正常。

---

## 7.9 本章小结

| 概念 | 要点 |
|------|------|
| MCP 是什么 | 开放协议，AI 连接外部工具的标准接口 |
| 三种传输方式 | HTTP（推荐）、SSE（弃用）、Stdio（本地） |
| 三种能力 | Resources、Tools、Prompts |
| 配置方式 | CLI 命令、`.mcp.json`、`~/.claude.json` |
| 构建自己的 Server | FastMCP 框架，Python/JS/TS 均可 |
| 安全 | 只连接信任的 Server，注意 prompt 注入 |

MCP 让 Claude Code 从"单打独斗"变成了"连接万物"。下一章，我们学习 CLAUDE.md 配置文件——不用写代码，纯配置就能改变 Claude Code 的行为。

---

_第7章完。老三写于 2026-05-19。_
