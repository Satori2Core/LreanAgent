# 018 · 15｜防微杜渐：Hooks 事件驱动自动化

> 📖 原文出处：[极客时间 - 黄佳《Claude Code 工程化实战》](https://time.geekbang.org/column/article/953022)
>
> 📅 学习时间：2026-07-05

---

## 这篇文章在回答什么问题？

1. **Hooks 是什么？** Claude Code 三大扩展机制中唯一能「拦截和修改」Claude 行为的机制——就像 AI 助手的中间件，在工具调用前后插入检查和处理。
2. **Hooks 能做什么？** 阻止危险命令、保护敏感文件、自动格式化代码、运行测试验证——安全防线的最后一道闸门。
3. **17 种 Hook 事件怎么分类？** 控制点（能阻止）、接管点（替代权限弹窗）、观察点（不能阻止只能记录）——三种能力等级。
4. **四种执行类型什么时候用？** Command（确定性脚本）> Prompt（LLM 评估）> Agent（子代理验证）> HTTP（远程服务）。

---

## 原文转述

### 一、Hooks 的本质：AI 时代的中间件

黄佳老师用 Web 中间件做类比：

```
HTTP 请求 → 中间件链 → 处理函数
Claude 调用 → [PreToolUse Hook] → 工具执行 → [PostToolUse Hook] → 响应
```

Web 中间件解决「业务代码不应该操心安全和日志」，Hooks 解决「Claude 不应该操心格式化和权限检查」——安全防线、质量守卫、审计日志全部自动完成。

**Commands / Skills / Hooks 三角**：

| | 回答的问题 | 机制 |
|------|----------|------|
| Commands | 做什么 | 手动触发任务 |
| Skills | 怎么做 | 知识注入 |
| **Hooks** | **能不能做** | **拦截和修改** |

> 💡 Hooks 是三者中唯一能**拦截和修改** Claude 行为的机制——你不止可以「放行或拒绝」，还可以「偷偷修改参数后执行」。

---

### 二、17 种 Hook 事件——三阵营

| 阵营 | 能力 | 代表事件 | 用途 |
|------|------|---------|------|
| **控制点** | 能阻止/修改 | PreToolUse, UserPromptSubmit, Stop | 拦截危险操作、拒绝不合理输入 |
| **接管点** | 替代默认行为 | PermissionRequest | 脚本自动批准/拒绝权限，替代人工 |
| **观察点** | 不能阻止只能记录 | PostToolUse, SessionStart, Notification | 格式化、审计日志、桌面通知 |

**设计原理**：工具执行前可以拦截（操作还没发生），执行后不能取消已写入的文件，但可以观察和反馈。

---

### 三、Hook 配置：三层嵌套

```json
{
  "hooks": {                    // 第一层：顶层容器
    "PreToolUse": [              // 第二层：事件类型（什么时候触发）
      {
        "matcher": "Bash",       // 第三层：匹配器（针对哪个工具）
        "hooks": [{               // 第三层：Hook 列表（执行什么）
          "type": "command",
          "command": "./hooks/block-dangerous.sh"
        }]
      }
    ]
  }
}
```

**配置位置**：
- 用户级 `~/.claude/settings.json` → 个人习惯
- 项目级 `.claude/settings.json` → 团队约定（提交到 git）
- 本地 `.claude/settings.local.json` → 临时覆盖
- 子代理 frontmatter → 子代理专属（仅在执行期间生效）

**Matcher 匹配**：精确（`"Write"`）、多工具（`"Edit|Write"`）、通配（`"*"`）、空（生命周期事件）。

---

### 四、四种执行类型

| 类型 | 能力 | 使用原则 |
|------|------|---------|
| **Command** | 确定性脚本执行 | 首选！正则匹配、格式检查 |
| **Prompt** | LLM 快速判断 | 需要语义理解但不需要翻文件 |
| **Agent** | 子代理验证 | 需要读文件、跑测试来确认 |
| **HTTP** | 远程服务 | 团队共享审计、集中式安全 |

> 💡 一句话原则：**能用 command 的不用 prompt，能用 prompt 的不用 agent**。确定性规则永远比 LLM 判断更可靠。

---

### 五、PreToolUse：工具执行前的守门员

PreToolUse 是唯一能**阻止**工具执行的 Hook 事件。它通过 stdin 接收 JSON 上下文（谁在执行、在哪执行、用什么工具、什么参数），通过退出码 + stdout JSON 返回决策。

**四种响应方式**：

```json
// 1. 允许执行
{"permissionDecision": "allow"}

// 2. 拒绝执行（只 exit 2 才是真正拦截）
{"permissionDecision": "deny", "permissionDecisionReason": "..."}

// 3. 交给用户确认
{"permissionDecision": "ask", "permissionDecisionReason": "..."}

// 4. 修改输入后执行（偷偷改参数！）
{"permissionDecision": "allow", "updatedInput": {"command": "..."}}
```

> 💡 **连续光谱**：allow → ask → deny，外加一个「暗中修正」的 updatedInput。优先选择最温和的响应——能 allow 不 ask，能 ask 不 deny。

---

### 六、PreToolUse 实战案例

**案例 1：阻止危险命令**——黑名单匹配 `rm -rf /`、`git push --force`、`DROP DATABASE` 等 15+ 种灾难性命令。用退出码 2（有意阻止）vs 退出码 1（脚本出错但不能阻止正常流）。

**核心调试技巧**：**调试信息必须输出到 stderr**（`>&2`），不能输出到 stdout。因为 stdout 被 Claude 用来读取 JSON 决策——这是 Hook 脚本开发最常见的坑。

**案例 2：保护敏感文件**——阻止 Claude 修改 `.env`、`credentials.json`、`*.pem` 等敏感文件。重点：不同工具传入不同字段——Bash 传 `command`，Write/Edit 传 `file_path`。

---

### 七、PostToolUse：工具执行后的质量守卫

不能阻止已发生的操作，但能做三件事：**后处理**（格式化）、**反馈**（向 Claude 注入 lint 结果）、**记录**（审计日志）。

**核心能力**：通过 `additionalContext` 向 Claude 注入反馈——Claude 看到「ESLint 发现 3 个错误」后会自动修复。这不是简单的日志，而是一个**闭环反馈机制**。

**实战案例：自动格式化**——每次 Write/Edit 后自动按文件扩展名选择格式化工具（Prettier/Black/gofmt/rustfmt）。

---

## 核心框架

### Hooks 三角定位

```
Commands（做什么）  Skills（怎么做）  Hooks（能不能做）
```

### 17 事件三阵营

```
控制点 = 能阻止（PreToolUse, UserPromptSubmit, Stop）
接管点 = 替代人工（PermissionRequest）
观察点 = 只能记录（PostToolUse, SessionStart, Notification...）
```

### 配置三层结构

```
事件类型 → 匹配器 → Hook 列表
```

### 执行四选一

```
command > prompt > agent > http
```

---

## 相关链接

- 📁 [原文原始数据](../article-origin/018/)
- 📖 [Anthropic Hooks 官方文档](https://code.claude.com/docs/en/hooks-guide)
- 🗂 [阅读指南](../000-阅读指南/)
