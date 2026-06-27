# 从零搭建 VS Code + Claude Code + DeepSeek：一篇搞定 AI 编程环境

> 记录从安装 VS Code、配置 Claude Code，到对接 DeepSeek 模型实现 AI 辅助编程的完整过程。

---

## 目录

1. [背景](#背景)
2. [环境准备：安装 VS Code](#环境准备安装-vs-code)
3. [安装 Claude Code](#安装-claude-code)
4. [获取 DeepSeek API Key](#获取-deepseek-api-key)
5. [配置 Claude Code 对接 DeepSeek](#配置-claude-code-对接-deepseek)
6. [实战使用](#实战使用)
7. [常见问题与排错](#常见问题与排错)
8. [总结](#总结)

---

## 背景

**Claude Code** 是 Anthropic 推出的终端级 AI 编程助手，原生对接 Claude 系列模型。但通过其自定义模型接口，我们也可以接入其他大语言模型（LLM），比如 **DeepSeek**。

**为什么选择 DeepSeek？**

- 💰 **性价比高**：DeepSeek API 定价远低于 Claude，适合个人开发者高频使用
- 🧠 **代码能力强**：DeepSeek-V3/V4 在编程基准测试中表现优异
- 🌏 **国内友好**：不需要翻墙即可直接访问
- ⚡ **响应速度快**：在国内网络环境下延迟更低

整个搭建过程分为四步：VS Code → Claude Code → DeepSeek API → 配置对接。

---

## 环境准备：安装 VS Code

### 第一步：下载 VS Code

前往 [VS Code 官网](https://code.visualstudio.com/) 下载安装包：

- **Windows**：下载 `.exe` 安装程序或 `.zip` 便携版
- **macOS**：下载 `.dmg` 或 `.zip`
- **Linux**：使用包管理器安装（推荐）

### 第二步：Windows 安装步骤

1. 双击下载的 `VSCodeSetup-x64-*.exe`
2. 接受许可协议，点击「下一步」
3. 选择安装路径（默认即可），点击「下一步」
4. **推荐勾选以下选项**：
   - ✅ 将 "通过 Code 打开" 添加到目录上下文菜单
   - ✅ 将 "通过 Code 打开" 添加到文件上下文菜单
   - ✅ 将 Code 注册为受支持的文件类型的编辑器
   - ✅ 添加到 PATH（重要！这样可以在终端直接用 `code` 命令）
5. 点击「安装」，完成后启动 VS Code

### 第三步：基础配置

安装完成后，推荐安装以下扩展（`Ctrl+Shift+X` 打开扩展面板）：

| 扩展名 | 用途 |
|--------|------|
| Chinese (Simplified) Language Pack | 中文界面 |
| GitLens | Git 增强 |
| Prettier | 代码格式化 |
| ESLint | JavaScript/TypeScript 代码检查 |
| Python | Python 开发支持 |

### 第四步：配置终端

在 VS Code 中打开终端（`` Ctrl+` ``），确认可以正常使用：

```bash
# 验证 code 命令已加入 PATH
code --version
```

> **提示**：VS Code 自带集成终端，后续 Claude Code 的安装和使用都将在这个终端中进行。

---

## 安装 Claude Code

### 方式一：npm 全局安装（推荐）

```bash
npm install -g @anthropic-ai/claude-code
```

### 方式二：直接使用 npx（免安装）

```bash
npx @anthropic-ai/claude-code
```

### 方式三：Windows 下的 Scoop 安装

```bash
scoop bucket add claude https://github.com/anthropics/claude-code-scoop.git
scoop install claude-code
```

### 验证安装

```bash
claude --version
```

如果输出类似 `Claude Code vX.X.X`，说明安装成功。

### 初次启动与认证

```bash
claude
```

首次启动时，Claude Code 会打开浏览器引导你完成 Anthropic 账号的 OAuth 认证。完成认证后，终端上就可以直接与 Claude 对话了。

> **注意**：认证后的默认模型走的是 Anthropic 官方的 Claude API，按量计费。但我们接下来要做的就是**将它指向 DeepSeek，使用更经济的模型**。

---

## 获取 DeepSeek API Key

### 第一步：注册 DeepSeek 账号

前往 [DeepSeek Platform](https://platform.deepseek.com/) 注册账号。

### 第二步：获取 API Key

1. 登录后进入 [API Keys 页面](https://platform.deepseek.com/api_keys)
2. 点击「创建 API Key」
3. 输入名称（如 `claude-code`），复制生成的 Key

> ⚠️ **重要**：API Key 只显示一次，请立即妥善保存！

### 第三步：充值（如需）

DeepSeek 新用户通常有一定的免费额度。如需充值，进入「充值」页面，国内支持支付宝/微信支付，门槛很低。

### DeepSeek 模型选择

| 模型 | 特点 | 推荐场景 |
|------|------|----------|
| `deepseek-chat` (V3) | 通用对话，速度快 | 日常编码辅助 |
| `deepseek-reasoner` (R1) | 深度推理，Chain-of-Thought | 复杂架构设计/调试 |
| `deepseek-v4-pro` | 最新旗舰，最强代码能力 | 高质量代码生成 |

---

## 配置 Claude Code 对接 DeepSeek

这是核心步骤。Claude Code 支持通过 `--model` 参数或配置文件指定自定义模型端点。

### 方案一：环境变量配置（推荐）

在你的 shell 配置文件（`~/.bashrc`、`~/.zshrc` 或 PowerShell Profile）中添加：

```bash
# DeepSeek API 配置
export ANTHROPIC_API_KEY="sk-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"

# 告诉 Claude Code 使用自定义 API 端点
export ANTHROPIC_BASE_URL="https://api.deepseek.com/v1"

# 指定默认使用的 DeepSeek 模型
export ANTHROPIC_MODEL="deepseek-v4-pro"
```

> **Windows 用户**：在 PowerShell 中设置环境变量：
>
> ```powershell
> $env:ANTHROPIC_API_KEY = "sk-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
> $env:ANTHROPIC_BASE_URL = "https://api.deepseek.com/v1"
> $env:ANTHROPIC_MODEL = "deepseek-v4-pro"
> ```
>
> 如果要永久设置，使用：
> ```powershell
> [Environment]::SetEnvironmentVariable("ANTHROPIC_API_KEY", "sk-xxx", "User")
> [Environment]::SetEnvironmentVariable("ANTHROPIC_BASE_URL", "https://api.deepseek.com/v1", "User")
> [Environment]::SetEnvironmentVariable("ANTHROPIC_MODEL", "deepseek-v4-pro", "User")
> ```

配置生效后，重新打开终端，启动 Claude Code：

```bash
claude
```

### 方案二：配置文件方式

创建 `~/.claude/settings.json`（或在项目根目录创建 `.claude/settings.json`）：

```json
{
  "apiKey": "sk-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
  "baseURL": "https://api.deepseek.com/v1",
  "model": "deepseek-v4-pro"
}
```

### 方案三：启动时指定模型

```bash
claude --model deepseek-chat
```

### 验证对接是否成功

启动 Claude Code 后，随便问一个问题：

```
> 请用 Python 写一个快速排序算法
```

如果正常返回代码，说明 DeepSeek 模型已经成功对接。

你也可以让 Claude Code 自报家门：

```
> 你是什么模型？
```

---

## 实战使用

### 常用命令

```bash
# 启动 Claude Code
claude

# 直接提问（单次模式）
claude "帮我分析这个项目的目录结构"

# 指定工作目录
claude --cwd /path/to/your/project

# 使用 DeepSeek Reasoner 进行深度推理
claude --model deepseek-reasoner "帮我分析这段代码的性能瓶颈"
```

### VS Code 集成

在 VS Code 中打开集成终端（`` Ctrl+` ``），直接输入 `claude` 即可在编辑器底部与 AI 对话，按 `Ctrl+Shift+` 新建终端窗口，可以实现：
- **左侧写代码 + 右侧 AI 对话** 的分屏布局
- **选中代码 → 终端中 `claude "解释这段代码"`** 的快捷工作流

### 典型工作场景

#### 1. 代码审查

```bash
claude "帮我 review 当前目录下的代码，找出潜在的问题和改进建议"
```

#### 2. 自动生成测试

```bash
claude "为 src/utils.py 中的每个函数生成 pytest 单元测试"
```

#### 3. 重构建议

```bash
claude "分析 app.js，给出模块拆分的重构方案"
```

#### 4. Bug 排查

```bash
claude "这个报错是什么原因？Traceback: ..."
```

### 使用 `.claude/settings.json` 做项目级配置

在项目根目录创建 `.claude/settings.json`，可以实现项目级别的定制：

```json
{
  "model": "deepseek-v4-pro",
  "allowedTools": ["Read", "Write", "Edit", "Bash", "Grep", "Glob"],
  "permissions": {
    "allow": [
      "Bash(npm test)",
      "Bash(npm run lint)"
    ]
  }
}
```

### 创建 CLAUDE.md 提升体验

在项目根目录创建 `CLAUDE.md`，写清楚项目规范，AI 每次都会自动读取：

```markdown
# 项目规范

- 使用 TypeScript 严格模式
- 函数命名使用 camelCase
- 每个函数必须有 JSDoc 注释
- 测试覆盖率要求 > 80%
- 提交信息遵循 Conventional Commits 规范
```

---

## 常见问题与排错

### Q1: Claude Code 报 "API key not found"

**原因**：未设置 `ANTHROPIC_API_KEY` 或环境变量未生效。

**解决**：
```bash
# 检查环境变量是否设置
echo $ANTHROPIC_API_KEY  # Linux/macOS/Git Bash

# 或者直接在启动时传入
claude --api-key sk-xxxxx
```

### Q2: 连接 DeepSeek 超时或报错

**可能原因**：
1. DeepSeek API 端点不对 — 确认是 `https://api.deepseek.com/v1`
2. 网络问题 — 国内用户网络通常不会有问题，可以 `curl https://api.deepseek.com/v1/models` 测试
3. API Key 余额不足 — 登录 DeepSeek 平台检查额度

### Q3: 部分功能不可用

需要注意：Claude Code 的某些高级功能（如 Artifact、Project 等）是 Anthropic 原生的，使用 DeepSeek 时可能会有部分限制。**基础的代码生成、审查、重构功能完全正常**。

### Q4: 响应速度慢

- 尝试切换模型：`deepseek-chat` (V3) 比 `deepseek-reasoner` (R1) 更快
- 避开高峰期使用（国内工作日晚间可能较慢）
- 确保网络连接稳定

### Q5: 上下文丢失 / 回答不连贯

DeepSeek 模型的上下文窗口和 Claude Code 的原生模型存在差异。如果出现回答不连贯的情况，可以：
- 使用 `/clear` 清空会话重新开始
- 减少单次提问中的文件数量
- 将大任务拆分成多个小步骤

### Q6: 每次启动都弹出登录方式选择，如何跳过？

**现象**：在终端输入 `claude` 后，每次都出现下面的交互式选择界面，必须手动选择才能继续：

```
Claude Code can be used with your Claude subscription or billed
based on API usage through your Console account.

Select login method:
> 1. Claude account with subscription · Pro, Max, Team, or Enterprise
  2. Anthropic Console account · API usage billing
  3. 3rd-party platform · Amazon Bedrock, Microsoft Foundry, or Vertex AI
```

这在 Windows 的 cmd / PowerShell / Git Bash 等终端中都会出现，不跳过就无法直接调用 CC。

**原因**：Claude Code 在启动时会检测你是否已完成认证。如果没检测到有效的认证信息（OAuth Token 或 API Key），就会弹出登录方式选择。使用 DeepSeek 时，我们不需要 Anthropic 的 OAuth 认证，但 CC 在检查到缺少认证状态时仍然会强制走这个流程。

**解决方案**：关键是用**环境变量直接注入 API 配置**，让 Claude Code 启动时检测到已有有效的 API 配置，从而跳过登录环节。需要同时设置以下三个环境变量：

#### Windows（PowerShell）

```powershell
# 设置为系统/用户级环境变量（推荐，一劳永逸）
[Environment]::SetEnvironmentVariable("ANTHROPIC_API_KEY", "sk-你的DeepSeek-api-key", "User")
[Environment]::SetEnvironmentVariable("ANTHROPIC_BASE_URL", "https://api.deepseek.com/v1", "User")
[Environment]::SetEnvironmentVariable("ANTHROPIC_MODEL", "deepseek-v4-pro", "User")

# 设置完后，关闭当前终端，重新打开，再试
claude
```

> ⚠️ **注意**：
> - 变量名是 `ANTHROPIC_API_KEY`（不是 `ANTHROPIC_API_KEY`），这是 Claude Code 原生识别的环境变量名
> - 设置「用户级」环境变量后，必须**重新打开终端**才能生效
> - 如果只是在当前终端用 `$env:XXX = "..."` 设置，关掉终端就失效了

#### Windows（cmd）

```cmd
setx ANTHROPIC_API_KEY "sk-你的DeepSeek-api-key"
setx ANTHROPIC_BASE_URL "https://api.deepseek.com/v1"
setx ANTHROPIC_MODEL "deepseek-v4-pro"
```

#### Linux / macOS / Git Bash

```bash
# 写入 shell 配置文件，永久生效
echo 'export ANTHROPIC_API_KEY="sk-你的DeepSeek-api-key"' >> ~/.bashrc
echo 'export ANTHROPIC_BASE_URL="https://api.deepseek.com/v1"' >> ~/.bashrc
echo 'export ANTHROPIC_MODEL="deepseek-v4-pro"' >> ~/.bashrc

# 立即生效
source ~/.bashrc
```

#### 方式二：使用 `.claude/settings.json`（项目级配置）

在用户目录或项目根目录创建 `.claude/settings.json`：

```json
{
  "apiKey": "sk-你的DeepSeek-api-key",
  "baseURL": "https://api.deepseek.com/v1",
  "model": "deepseek-v4-pro"
}
```

#### 验证是否跳过成功

设置完成后，在终端直接输入：

```bash
claude
```

如果能直接进入对话界面（看到 `>` 提示符），不再弹出登录方式选择，就说明配置成功了。

> 💡 **补充说明**：那个登录界面的选项 3（3rd-party platform）目前只列出了 Amazon Bedrock、Microsoft Foundry、Vertex AI，并没有 DeepSeek。所以选了 3 也没用，正确的做法就是通过环境变量/配置文件注入，直接绕过这个步骤。

---

## 总结

至此，你已经完成了从零搭建 VS Code + Claude Code + DeepSeek 的全过程：

```
VS Code 编辑器  →  Claude Code 终端助手  →  DeepSeek 模型引擎
  (写代码)           (AI 驱动)                (经济、快速、强大)
```

**核心优势回顾**：

| 维度 | 说明 |
|------|------|
| 💸 **成本** | DeepSeek API 价格约为 Claude 的 1/10 ~ 1/20 |
| 🚀 **速度** | 国内网络直连，延迟显著低于海外 API |
| 🔧 **体验** | Claude Code 的终端交互体验 + DeepSeek 的代码能力 |
| 🆓 **门槛** | 全部工具均可免费开始使用 |

希望这篇博客能帮助你顺利完成环境搭建。如果在配置过程中遇到任何问题，欢迎在评论区交流！

---

> **更新记录**：2026-06-27 首次发布
>
> **适用版本**：VS Code 1.9x+ | Claude Code 最新版 | DeepSeek API (deepseek-v4-pro / deepseek-chat)
