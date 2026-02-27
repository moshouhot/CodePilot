# CodePilot 完整使用教程

> CodePilot 是 Claude Code 的原生桌面 GUI 客户端，让你通过可视化界面与 Claude AI 进行对话、编码和项目管理，无需在终端中操作。

---

## 目录

- [一、安装与配置](#一安装与配置)
  - [1.1 环境要求](#11-环境要求)
  - [1.2 安装方式](#12-安装方式)
  - [1.3 首次启动配置](#13-首次启动配置)
- [二、API 配置（重点）](#二api-配置重点)
  - [2.1 配置 API Provider](#21-配置-api-provider)
  - [2.2 支持的 API 提供商](#22-支持的-api-提供商)
  - [2.3 自定义 API 端点](#23-自定义-api-端点)
  - [2.4 环境变量配置](#24-环境变量配置)
- [三、核心功能使用](#三核心功能使用)
  - [3.1 创建新会话](#31-创建新会话)
  - [3.2 对话交互](#32-对话交互)
  - [3.3 工作目录设置](#33-工作目录设置)
  - [3.4 交互模式切换](#34-交互模式切换)
  - [3.5 模型选择](#35-模型选择)
  - [3.6 权限控制](#36-权限控制)
- [四、扩展功能](#四扩展功能)
  - [4.1 MCP 服务器管理](#41-mcp-服务器管理)
  - [4.2 自定义技能（Skills）](#42-自定义技能skills)
  - [4.3 文件附件](#43-文件附件)
- [五、设置与配置](#五设置与配置)
  - [5.1 Claude CLI 设置](#51-claude-cli-设置)
  - [5.2 自动审批模式](#52-自动审批模式)
- [六、常见问题](#六常见问题)
- [七、开发者指南](#七开发者指南)

---

## 一、安装与配置

### 1.1 环境要求

| 要求 | 最低版本 |
|------|---------|
| **Node.js** | 18+ |
| **Claude Code CLI** | 已安装并完成认证 |
| **npm** | 9+（Node 18 自带） |

> **重要提示**：CodePilot 底层调用 Claude Code Agent SDK，因此需要确保 `claude` 命令在系统 PATH 中可用，并且已完成认证（`claude login`）。

### 1.2 安装方式

#### 方式一：下载预编译版本（推荐）

前往 [Releases](https://github.com/op7418/CodePilot/releases) 页面下载对应平台的安装包：

- **macOS**：下载 `.dmg` 文件，支持 Apple Silicon (arm64) 和 Intel (x64)
- **Windows**：下载 `.zip` 文件，解压后直接运行
- **Linux**：下载 `.AppImage`、`.deb` 或 `.rpm` 格式

#### 方式二：从源码构建

```bash
# 克隆仓库
git clone https://github.com/op7418/CodePilot.git
cd CodePilot

# 安装依赖
npm install

# 开发模式运行
npm run dev              # 仅 Next.js（浏览器访问 http://localhost:3000）
npm run electron:dev     # 完整 Electron 桌面应用

# 生产构建
npm run electron:pack    # 打包为可分发应用
```

### 1.3 首次启动配置

#### macOS 安全提示处理

由于应用未进行代码签名，首次打开会提示"无法验证开发者"：

1. **右键打开**：在访达中右键点击 `CodePilot.app`，选择"打开"
2. **系统设置**：前往 **系统设置 > 隐私与安全性**，点击"仍要打开"
3. **终端命令**：`xattr -cr /Applications/CodePilot.app`

#### Windows SmartScreen 处理

1. 点击"更多信息"
2. 点击"仍要运行"

---

## 二、API 配置（重点）

CodePilot 支持灵活的 API 配置，你可以使用 Anthropic 官方 API、第三方代理或自建端点。

### 2.1 配置 API Provider

1. 打开 CodePilot，点击左侧导航栏的 **Settings**（齿轮图标）
2. 找到 **API Providers** 区域
3. 点击 **Add Provider** 或使用快速预设按钮

#### Provider 配置字段说明

| 字段 | 说明 | 示例 |
|------|------|------|
| **Name** | 自定义名称，方便识别 | `My Anthropic` |
| **Provider Type** | 提供商类型 | `anthropic` / `openrouter` / `custom` |
| **Base URL** | API 端点地址 | `https://api.anthropic.com` |
| **API Key** | API 密钥 | `sk-ant-...` |
| **Extra Env** | 额外环境变量（JSON 格式） | `{"API_TIMEOUT_MS":"3000000"}` |
| **Notes** | 备注说明 | 可选 |

### 2.2 支持的 API 提供商

CodePilot 内置了多个快速预设：

| 提供商 | Base URL | 说明 |
|--------|----------|------|
| **Anthropic** | `https://api.anthropic.com` | 官方 API |
| **OpenRouter** | `https://openrouter.ai/api` | 多模型聚合 |
| **GLM (中国)** | `https://open.bigmodel.cn/api/anthropic` | 智谱 AI 中国区 |
| **GLM (Global)** | `https://api.z.ai/api/anthropic` | 智谱 AI 国际区 |
| **Kimi Coding Plan** | `https://api.kimi.com/coding/` | 月之暗面 Kimi |
| **Moonshot** | `https://api.moonshot.cn/anthropic` | 月之暗面 |
| **MiniMax (中国)** | `https://api.minimaxi.com/anthropic` | MiniMax 中国区 |
| **MiniMax (Global)** | `https://api.minimax.io/anthropic` | MiniMax 国际区 |
| **AWS Bedrock** | - | 需要 AWS 凭证 |
| **Google Vertex** | - | 需要 GCP 凭证 |
| **LiteLLM** | `http://localhost:4000` | 本地代理 |

### 2.3 自定义 API 端点

如果你使用自建的 API 代理（如 one-api、new-api 等），按以下步骤配置：

1. 点击 **Add Provider**
2. 填写配置：
   ```
   Name: My Proxy
   Provider Type: custom
   Base URL: https://your-proxy.com/v1
   API Key: your-api-key
   ```
3. 在 **Extra Env** 中添加额外配置（如需要）：
   ```json
   {
     "API_TIMEOUT_MS": "3000000",
     "ANTHROPIC_API_KEY": ""
   }
   ```

   > **注意**：`ANTHROPIC_API_KEY: ""` 表示清除该环境变量，用于避免与 `ANTHROPIC_AUTH_TOKEN` 冲突

4. 点击保存，然后点击 **Apply** 激活该提供商

### 2.4 环境变量配置

如果你不想在 UI 中配置，也可以通过环境变量设置：

```bash
# 设置 API Key（二选一）
export ANTHROPIC_API_KEY="sk-ant-..."
# 或
export ANTHROPIC_AUTH_TOKEN="sk-ant-..."

# 设置自定义 API 端点
export ANTHROPIC_BASE_URL="https://your-proxy.com/v1"
```

> **优先级说明**：UI 中激活的 Provider 会覆盖环境变量配置。如果想使用环境变量，请确保没有激活任何 Provider。

---

## 三、核心功能使用

### 3.1 创建新会话

1. 点击左侧导航栏的 **Chat** 图标（或首页的"New Chat"）
2. 进入新会话页面
3. 选择工作目录（项目路径）
4. 开始对话

### 3.2 对话交互

#### 发送消息

- 在底部输入框输入消息
- 按 `Enter` 发送，`Shift+Enter` 换行
- 支持 Markdown 格式

#### 附加文件

- 点击输入框左侧的附件图标
- 支持图片（直接预览）和其他文件（保存后通过 Read 工具读取）

#### 内置斜杠命令

| 命令 | 说明 |
|------|------|
| `/help` | 显示帮助信息 |
| `/clear` | 清除当前会话历史 |
| `/cost` | 显示 Token 用量和费用 |
| `/compact` | 压缩上下文 |
| `/doctor` | 诊断环境配置 |
| `/review` | 代码审查 |

### 3.3 工作目录设置

每个会话都可以指定独立的工作目录：

1. 在会话页面顶部的 Header 区域，点击项目路径
2. 选择新的工作目录
3. 右侧面板会显示该目录的文件树

Claude 会基于该目录进行文件操作和代码理解。

### 3.4 交互模式切换

CodePilot 支持三种交互模式：

| 模式 | 说明 |
|------|------|
| **Code** | 默认模式，Claude 可以读写文件、执行命令 |
| **Plan** | 规划模式，Claude 只提供计划不执行操作 |
| **Ask** | 问答模式，Claude 只回答问题不操作文件 |

在 Header 区域点击模式按钮进行切换。

### 3.5 模型选择

支持在对话中切换 Claude 模型：

- **claude-sonnet-4-20250514**（默认）
- **claude-opus-4-20250514**
- **claude-3-5-haiku-20241022**

点击 Header 区域的模型选择器进行切换。

### 3.6 权限控制

当 Claude 尝试执行工具操作时，CodePilot 会显示权限请求：

- **Allow**：允许本次操作
- **Allow Always**：允许该工具的所有操作
- **Deny**：拒绝本次操作

---

## 四、扩展功能

### 4.1 MCP 服务器管理

MCP（Model Context Protocol）让你可以扩展 Claude 的能力。

#### 添加 MCP 服务器

1. 点击左侧导航栏的 **Extensions** 图标
2. 选择 **MCP Servers** 标签
3. 点击 **Add Server**
4. 配置服务器：

**stdio 类型**（本地命令）：
```json
{
  "command": "npx",
  "args": ["-y", "@anthropic-ai/mcp-server-sqlite"],
  "env": {
    "DATABASE_PATH": "/path/to/db.sqlite"
  }
}
```

**sse 类型**（服务端事件）：
```json
{
  "type": "sse",
  "url": "http://localhost:8080/sse",
  "headers": {
    "Authorization": "Bearer token"
  }
}
```

**http 类型**：
```json
{
  "type": "http",
  "url": "http://localhost:8080/mcp",
  "headers": {}
}
```

### 4.2 自定义技能（Skills）

技能是可复用的提示词模板，可以作为斜杠命令调用。

#### 创建技能

1. 进入 **Extensions > Skills**
2. 点击 **Add Skill**
3. 填写：
   - **Name**：技能名称（如 `review`）
   - **Description**：技能描述
   - **Prompt**：提示词模板

#### 使用技能

在聊天中输入 `/技能名称` 即可调用，例如 `/review` 会触发代码审查。

### 4.3 文件附件

支持在对话中附加文件：

- **图片文件**：直接以多模态方式发送给 Claude（支持 jpg、png、gif、webp）
- **其他文件**：保存到 `.codepilot-uploads/` 目录，Claude 通过 Read 工具访问

---

## 五、设置与配置

### 5.1 Claude CLI 设置

CodePilot 可以编辑 `~/.claude/settings.json`：

1. 进入 **Settings** 页面
2. 使用 **Visual Editor** 或 **JSON Editor**
3. 可配置的选项：
   - **Permissions**：权限规则
   - **Environment Variables**：环境变量

### 5.2 自动审批模式

> ⚠️ **危险功能**：启用后 Claude 将自动执行所有操作，无需确认

1. 进入 **Settings** 页面
2. 找到 **Auto-approve All Actions** 开关
3. 启用后，所有工具操作将自动批准

适用场景：
- 完全信任的自动化任务
- 批量处理操作
- 自动化测试

---

## 六、常见问题

### Q1: 无法连接 API

**检查步骤**：
1. 确认 API Key 正确
2. 确认 Base URL 格式正确（注意末尾是否需要 `/v1`）
3. 检查网络连接
4. 查看 `~/.codepilot/` 目录下的日志

### Q2: Claude CLI 未找到

**解决方案**：
```bash
# 确认 Claude CLI 已安装
claude --version

# 如果未安装
npm install -g @anthropic-ai/claude-code

# 完成认证
claude login
```

### Q3: 数据存储位置

| 类型 | 路径 |
|------|------|
| 数据库 | `~/.codepilot/codepilot.db` |
| Claude 设置 | `~/.claude/settings.json` |
| MCP 配置 | `~/.claude/claude.config.json` |

### Q4: Windows 上的兼容性

CodePilot 会自动检测 Git Bash 并配置 `CLAUDE_CODE_GIT_BASH_PATH`。如果遇到问题，可手动设置：

```bash
set CLAUDE_CODE_GIT_BASH_PATH=C:\Program Files\Git\bin\bash.exe
```

---

## 七、开发者指南

### 项目结构

```
codepilot/
├── electron/                # Electron 主进程
│   ├── main.ts              # 窗口创建、服务器生命周期
│   └── preload.ts           # Context bridge
├── src/
│   ├── app/                 # Next.js 页面和 API
│   │   ├── chat/            # 对话页面
│   │   ├── extensions/      # 扩展管理
│   │   ├── settings/        # 设置页面
│   │   └── api/             # REST API
│   ├── components/          # React 组件
│   ├── hooks/               # 自定义 Hooks
│   ├── lib/                 # 核心逻辑
│   │   ├── claude-client.ts # SDK 封装
│   │   ├── db.ts            # SQLite 数据库
│   │   └── platform.ts      # 平台适配
│   └── types/               # TypeScript 类型
└── electron-builder.yml     # 打包配置
```

### 技术栈

- **框架**：Next.js 16 + Electron 40
- **UI**：Radix UI + shadcn/ui + Tailwind CSS 4
- **AI**：Claude Agent SDK
- **数据库**：better-sqlite3（WAL 模式）
- **测试**：Playwright

### 开发命令

```bash
npm run dev              # 开发模式（浏览器）
npm run electron:dev     # 开发模式（桌面）
npm run build            # 生产构建
npm run electron:pack    # 打包应用
npm run lint             # 代码检查
```

---

## 反馈与支持

- **GitHub Issues**：[提交问题](https://github.com/op7418/CodePilot/issues)
- **贡献代码**：Fork 仓库，提交 Pull Request

---

**许可证**：MIT

**作者**：op7418
