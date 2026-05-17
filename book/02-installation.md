# 第2章：环境搭建与安装

> 工欲善其事，必先利其器。本章带你完成 Claude Code SDK 的安装和配置，为后续学习打下坚实基础。

## 2.1 系统要求

在安装 Claude Code SDK 之前，请确保你的系统满足以下要求：

### 2.1.1 支持的操作系统

| 操作系统 | 支持状态 | 备注 |
|---------|---------|------|
| macOS | ✅ 完全支持 | 推荐使用 |
| Linux | ✅ 完全支持 | 主流发行版均可 |
| Windows | ✅ 完全支持 | Windows 10/11 |
| WSL | ✅ 完全支持 | Windows Subsystem for Linux |

### 2.1.2 硬件要求

- **内存**：建议 8GB 以上
- **存储**：至少 500MB 可用空间
- **网络**：需要稳定的互联网连接（用于调用 Claude API）

### 2.1.3 软件依赖

虽然 Claude Code SDK 可以独立运行，但以下软件能增强其功能：

- **Git**：用于代码版本控制集成
- **Node.js 18+**：如果你要使用 SDK 编程接口
- **现代终端**：支持 ANSI 颜色的终端应用

### 2.1.4 账户要求

使用 Claude Code SDK 需要以下账户之一：

- **Claude Pro/Max**：个人订阅用户
- **Claude Teams**：团队订阅用户
- **Claude Enterprise**：企业订阅用户
- **Anthropic Console API Key**：API 按量付费用户

> 💡 **提示**：如果你还没有 Claude 账户，可以访问 [claude.ai](https://claude.ai) 注册。免费账户无法使用 Claude Code SDK。

---

## 2.2 安装方式

Claude Code SDK 提供了多种安装方式，根据你的操作系统和偏好选择：

### 2.2.1 macOS / Linux 安装（推荐）

**方式一：官方安装脚本（推荐）**

这是官方最推荐的安装方式，会自动检测系统并完成安装：

```bash
# 一键安装
curl -fsSL https://claude.ai/install.sh | bash
```

安装脚本会：
1. 检测你的操作系统和架构
2. 下载最新版本的 Claude Code 二进制文件
3. 安装到 `/usr/local/bin/claude`（或 `~/.local/bin/claude`）
4. 配置必要的权限

**方式二：Homebrew 安装**

如果你使用 Homebrew 包管理器：

```bash
# 添加 Anthropic 的 tap
brew tap anthropics/claude-code

# 安装 Claude Code
brew install --cask claude-code
```

Homebrew 方式的优势：
- 自动处理依赖
- 方便升级：`brew upgrade claude-code`
- 方便卸载：`brew uninstall claude-code`

### 2.2.2 Windows 安装

**方式一：PowerShell 脚本（推荐）**

在 PowerShell（以管理员身份运行）中执行：

```powershell
# 一键安装
irm https://claude.ai/install.ps1 | iex
```

安装脚本会：
1. 下载 Claude Code Windows 版本
2. 安装到用户目录或 Program Files
3. 自动添加到 PATH 环境变量

**方式二：WinGet 安装**

Windows 11 内置的包管理器：

```powershell
# 使用 WinGet 安装
winget install Anthropic.ClaudeCode
```

### 2.2.3 NPM 安装（已弃用）

⚠️ **重要提示**：npm 安装方式已被官方标记为**弃用（deprecated）**，不推荐用于生产环境。

```bash
# 不推荐使用此方式
npm install -g @anthropic-ai/claude-code
```

**npm 方式的问题：**
- 启动速度较慢（需要 Node.js 运行时）
- 版本更新可能滞后
- 官方已不再积极维护此安装方式

但如果你的项目需要 SDK 编程接口（TypeScript/JavaScript API），仍需安装 npm 包：

```bash
# 作为项目依赖安装（用于 SDK 编程）
npm install @anthropic-ai/claude-code
# 或
pnpm add @anthropic-ai/claude-code
```

---

## 2.3 验证安装

安装完成后，验证是否正确安装：

### 2.3.1 检查版本

```bash
# 查看版本号
claude --version

# 预期输出示例：
# claude-code version 2.1.143
```

### 2.3.2 检查安装路径

```bash
# 查看 claude 命令的位置
which claude  # macOS/Linux
where claude  # Windows

# 预期输出示例：
# /usr/local/bin/claude
```

### 2.3.3 运行测试命令

```bash
# 快速测试 Claude Code 是否正常工作
claude -p "输出 Hello World"
```

如果一切正常，Claude Code 会响应并输出结果。

---

## 2.4 身份验证

首次使用 Claude Code SDK 需要进行身份验证。

### 2.4.1 交互式登录（推荐）

最简单的验证方式是交互式登录：

```bash
# 启动 Claude Code
claude

# 首次启动会提示你登录
# > Please login to continue...
# > Press Enter to open browser...
```

按回车后，系统会：
1. 打开浏览器访问 Anthropic 登录页面
2. 要求你登录 Claude 账户
3. 完成授权后自动返回终端

### 2.4.2 API Key 验证

如果你使用 Anthropic Console 的 API Key：

```bash
# 设置环境变量
export ANTHROPIC_API_KEY="your-api-key-here"

# 或在 Claude Code 中交互式输入
claude
# > Choose authentication method:
# > 1. Claude.ai account (requires subscription)
# > 2. Anthropic Console API Key
# 选择 2，然后粘贴你的 API Key
```

**API Key 的优势：**
- 按实际使用量付费
- 适合自动化脚本和 CI/CD 环境
- 可以设置预算限制

### 2.4.3 查看认证状态

```bash
# 查看当前认证状态
claude /status

# 输出示例：
# Authentication: Claude Pro (claude.ai)
# Model: claude-sonnet-4-20250514
```

---

## 2.5 配置文件

Claude Code SDK 使用多个配置文件来管理行为。

### 2.5.1 全局配置目录

配置文件位于以下目录：

```
~/.claude/                     # 全局配置目录
├── CLAUDE.md                  # 全局提示词（所有项目共享）
├── settings.json              # 全局设置
├── credentials.json           # 认证信息（自动生成）
└── projects/                  # 项目缓存
    └── project-name/
        ├── CLAUDE.md          # 项目级提示词
        └── CLAUDE.local.md    # 本地级提示词
```

### 2.5.2 全局设置文件

`~/.claude/settings.json` 用于全局配置：

```json
{
  "theme": "dark",
  "defaultModel": "claude-sonnet-4-20250514",
  "outputFormat": "text",
  "autoUpdate": true,
  "telemetry": {
    "enabled": true
  }
}
```

### 2.5.3 环境变量配置

Claude Code SDK 支持以下环境变量：

| 环境变量 | 说明 | 默认值 |
|---------|------|--------|
| `ANTHROPIC_API_KEY` | API Key | - |
| `ANTHROPIC_MODEL` | 默认模型 | `claude-sonnet-4-20250514` |
| `CLAUDE_CODE_MAX_TOKENS` | 最大输出 Token | 4096 |
| `CLAUDE_CODE_TIMEOUT` | 请求超时（秒） | 300 |
| `CLAUDE_CODE_DEBUG` | 启用调试模式 | `false` |

使用示例：

```bash
# 在 ~/.zshrc 或 ~/.bashrc 中添加
export ANTHROPIC_API_KEY="sk-ant-..."
export CLAUDE_CODE_MAX_TOKENS=8192
```

---

## 2.6 升级与卸载

### 2.6.1 升级 Claude Code

Claude Code SDK 更新频繁，建议定期升级：

**curl 脚本安装：**

```bash
# 重新运行安装脚本即可升级
curl -fsSL https://claude.ai/install.sh | bash
```

**Homebrew 安装：**

```bash
brew update && brew upgrade claude-code
```

**WinGet 安装：**

```powershell
winget upgrade Anthropic.ClaudeCode
```

**npm 安装：**

```bash
npm update -g @anthropic-ai/claude-code
```

### 2.6.2 卸载 Claude Code

如果需要卸载 Claude Code：

**curl 脚本安装：**

```bash
# 删除二进制文件
sudo rm /usr/local/bin/claude
# 或
rm ~/.local/bin/claude

# 删除配置（可选）
rm -rf ~/.claude
```

**Homebrew 安装：**

```bash
brew uninstall claude-code
```

**WinGet 安装：**

```powershell
winget uninstall Anthropic.ClaudeCode
```

---

## 2.7 常见安装问题

### 2.7.1 权限错误

**问题**：安装时提示 "Permission denied"

```bash
# 错误示例
curl: Failed to create file '/usr/local/bin/claude': Permission denied
```

**解决方案**：

```bash
# 方式一：使用 sudo
sudo curl -fsSL https://claude.ai/install.sh | bash

# 方式二：安装到用户目录
mkdir -p ~/.local/bin
curl -fsSL https://claude.ai/install.sh | INSTALL_DIR=~/.local/bin bash
```

### 2.7.2 网络问题

**问题**：无法下载安装脚本

**解决方案**：

```bash
# 使用国内镜像（如果可用）
# 或设置代理
export https_proxy=http://127.0.0.1:7890
curl -fsSL https://claude.ai/install.sh | bash
```

### 2.7.3 命令未找到

**问题**：安装成功但 `claude` 命令找不到

```bash
claude: command not found
```

**解决方案**：

```bash
# 检查 PATH 是否包含安装目录
echo $PATH

# 如果 ~/.local/bin 不在 PATH 中，添加到 shell 配置
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

### 2.7.4 认证失败

**问题**：登录后仍提示未认证

**解决方案**：

```bash
# 清除认证缓存
rm ~/.claude/credentials.json

# 重新登录
claude /login
```

---

## 2.8 IDE 集成

除了命令行，Claude Code SDK 还可以集成到 IDE 中。

### 2.8.1 VS Code 集成

```bash
# 安装 VS Code 扩展
code --install-extension anthropic.claude-code
```

安装后，VS Code 侧边栏会出现 Claude 图标，点击即可使用。

**功能特点：**
- 代码差异预览
- 一键接受/拒绝修改
- 文件上下文自动关联
- 快捷键支持

### 2.8.2 JetBrains 集成

在 JetBrains IDE（IntelliJ IDEA、PyCharm、WebStorm 等）中：

1. 打开 Settings → Plugins → Marketplace
2. 搜索 "Claude Code"
3. 点击 Install

**功能特点：**
- 与 JetBrains VCS 集成
- 代码审查辅助
- 重构建议

---

## 2.9 实践：完整安装流程

下面是一个完整的安装和验证流程示例：

```bash
#!/bin/bash
# Claude Code SDK 完整安装脚本

echo "=== Claude Code SDK 安装脚本 ==="

# 1. 检查系统
echo "1. 检查系统..."
uname -a

# 2. 安装 Claude Code
echo "2. 安装 Claude Code..."
curl -fsSL https://claude.ai/install.sh | bash

# 3. 验证安装
echo "3. 验证安装..."
claude --version

# 4. 检查认证状态
echo "4. 检查认证状态..."
claude /status

# 5. 运行测试
echo "5. 运行测试..."
claude -p "用一句话解释什么是 Claude Code SDK"

echo "=== 安装完成 ==="
```

运行结果示例：

```
=== Claude Code SDK 安装脚本 ===
1. 检查系统...
Darwin MacBook-Pro.local 23.0.0 Darwin Kernel Version 23.0.0
2. 安装 Claude Code...
Downloading claude-code-2.1.143-darwin-x64...
Installation complete!
3. 验证安装...
claude-code version 2.1.143
4. 检查认证状态...
Authentication: Not logged in
5. 运行测试...
Claude Code SDK is an AI-powered coding assistant...
=== 安装完成 ===
```

---

## 2.10 总结

本章我们学习了：

- **系统要求**：支持 macOS、Linux、Windows，需要 Claude 账户
- **安装方式**：curl 脚本（推荐）、Homebrew、PowerShell、WinGet
- **身份验证**：交互式登录或 API Key
- **配置文件**：全局配置、环境变量、项目配置
- **升级卸载**：定期升级保持最新版本
- **IDE 集成**：VS Code、JetBrains 插件

安装完成后，你已经可以开始使用 Claude Code SDK 了。下一章，我们将编写第一个 SDK 程序，体验 AI 编程助手的强大能力。

---

## 参考资料

1. [Claude Code GitHub 仓库](https://github.com/anthropics/claude-code)
2. [Claude Code 官方文档](https://code.claude.com/docs)
3. [Anthropic Console](https://console.anthropic.com)
4. [NPM 包信息](https://www.npmjs.com/package/@anthropic-ai/claude-code)

---

**下一章预告**：第3章将带你编写第一个 SDK 程序，从 Hello World 开始，逐步理解 SDK 的核心流程。
