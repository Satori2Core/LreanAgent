# 020 · 17｜集腋成裘：MCP 协议与外部工具集成

> 📖 原文出处：[极客时间 - 黄佳《Claude Code 工程化实战》](https://time.geekbang.org/column/article/955015)
>
> 📅 学习时间：2026-07-12

---

## 这篇文章在回答什么问题？

1. **Claude Code 的能力边界在哪？** Memory、SubAgents、Skills、Commands、Hooks——全部局限于本地文件系统。数据库、GitHub、Notion、Sentry 这些外部系统，Claude 怎么连接？
2. **MCP 是什么？** Model Context Protocol——AI 的 USB-C 接口。把 M×N 的碎片化集成问题变成 M+N，一个协议通用连接。
3. **怎么用 MCP 给 Claude Code 接外部服务？** 三种传输方式（stdio/HTTP/SSE）、配置管理（`claude mcp add`）、安全凭证处理。

---

## 原文转述

### 一、M×N → M+N：MCP 解决的问题

在 MCP 出现前，M 个 AI 助手 × N 个外部服务 = M×N 个专用适配器。每对组合都要单独开发维护。

MCP 的洞察：**一个协议，通用连接**。每个 AI 助手只实现一次 MCP Client，每个服务只实现一次 MCP Server，任意组合即可工作。

2024 年 11 月 Anthropic 发布 → 2025 年 3 月 OpenAI 采纳 → 2025 年 9 月 Google 支持 → 2025 年 12 月捐赠给 AAIF。目前 9700 万月下载、10000+ 公开 MCP 服务器。

---

### 二、架构：类 LSP 的客户端-服务器模型

MCP 复用了 Language Server Protocol 的设计思想——Claude Code 是 Client，MCP Server 暴露三种能力：

| 能力 | 用途 | 类比 |
|------|------|------|
| **Tools** | 让 Claude「做事情」 | 函数调用 |
| **Resources** | 让 Claude「看到东西」 | 只读数据源 |
| **Prompts** | 预定义场景交互模板 | 快捷操作 |

Claude Code 启动时自动发现所有配置的 MCP Server 及其能力。用户说「帮我查数据库用户数」，Claude 自动找到数据库 MCP Server → 调用查询工具 → 解析返回。全程对用户透明。

---

### 三、三种传输方式

| 方式 | 适用 | 配置 |
|------|------|------|
| **stdio** | 本地进程 | `"command": "npx", "args": [...]` |
| **HTTP** | 远程服务（推荐） | `"url": "https://...", "headers": {...}` |
| **SSE** | 实时推送（少用） | 类似 HTTP，持久连接 |

原则：**本地用 stdio，远程用 HTTP，实时用 SSE**。前两者覆盖 95% 场景。

---

### 四、配置与管理

配置文件独立于 `settings.json`：

```
项目级：.mcp.json（提交到 git，团队共享）
用户级：~/.claude.json 顶级的 mcpServers
本地级：~/.claude.json projects.<path>.mcpServers（不入 git）
```

命令行管理：
```bash
claude mcp add --transport http github https://api.githubcopilot.com/mcp/
claude mcp add filesystem -- npx @modelcontextprotocol/server-filesystem /path
claude mcp list
claude mcp remove github
```

敏感信息用 `${ENV_VAR}` 引用，不要硬编码。

---

## 核心框架

### MCP 核心公式

```
M×N（碎片化）→ M+N（标准化）= 一个协议通用连接
```

### 三种传输

```
本地 stdio | 远程 HTTP | 实时 SSE
```

### 三种能力

```
Tools（做）| Resources（看）| Prompts（模板）
```

---

## 相关链接

- 📁 [原文原始数据](../article-origin/020/)
- 📖 [MCP 官方文档](https://modelcontextprotocol.io/)
- 🗂 [阅读指南](../000-阅读指南/)
