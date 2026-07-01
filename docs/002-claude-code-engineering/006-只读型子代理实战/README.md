# 006 · 05｜实战：创建只读型和安全型子代理

> 📖 原文出处：[极客时间 - 黄佳《Claude Code 工程化实战》](https://time.geekbang.org/column/article/944205)
>
> 📅 学习时间：2026-07-01

---

## 这篇文章在回答什么问题？

1. **怎么从零创建一个能用的 Sub-Agent？** 前三篇讲了概念和模式，这篇终于动手了——创建代码审查员子代理，完整走一遍「痛点→设计→配置→运行→验证」的闭环。
2. **权限边界怎么从「希望如此」变成「物理上做不到」？** 只读型子代理不给 Edit/Write 权限——不是软约束，是系统级硬边界。
3. **SubAgent + Skill 怎么组合使用？** 通过 `skills` 字段预加载领域知识（链路拓扑、历史事故），让子代理「在开始工作前就带着知识」。
4. **到底什么时候该创建子代理？** 不是每次都用，也不是从来不用——有一个实用的决策框架。

---

## 原文转述

### 一、为什么从「只读型」开始？

黄佳老师开场用一个场景直击要害：

```
你在主对话中说：「帮我检查一下 auth.js 的安全性」
Claude 读了代码 → 发现了硬编码密钥 → 顺手帮你改成了环境变量读取

等等！
你只是让它看看，没让它改啊！
而且它的「好心修复」可能引入了新问题——
比如你根本不想配环境变量。
```

这就是为什么需要只读型子代理：**审查者应该只有读取能力，绝对不能修改代码。**

```
代码审查员的职责边界：
✅ 可以做：读取代码、分析问题、输出报告
❌ 不可做：修改文件、执行可能有副作用的命令
```

---

### 二、痛点驱动的设计思维

在动手写配置之前，先问一个方法论级别的问题：**我们为什么要创建这个子代理？**

黄佳老师给出了一套「痛点反推设计」的思维框架：

```
工程痛点 → 分析缺什么能力 → 设计职责边界 → 选择工具组合 → 配置子代理
```

具体来说，面对一个「AI 做得不够好」的场景，依次回答：

| 步骤 | 问题 | 示例（代码审查场景） |
|------|------|-------------------|
| 1. 识别痛点 | AI 哪里做得不对？ | AI 审查代码时会顺手修改 |
| 2. 分析缺什么 | 缺少什么机制？ | 缺少「只读不写」的权限边界 |
| 3. 设计职责 | 边界在哪？ | 能读代码、分析问题、出报告；不能改文件 |
| 4. 选工具 | 需要什么工具？ | Read, Grep, Glob（不要 Edit/Write） |
| 5. 配子代理 | 配置怎么写？ | frontmatter + system prompt |

> 💡 这套框架比任何一份具体配置都重要。掌握了它，你能面对任何工程场景设计出合适的子代理。

---

### 三、第一步：理解「有问题」的示例代码

课程提供的示例代码故意包含安全问题。摘几个典型的：

**auth.js 的问题**：
- 硬编码密钥（`SECRET_KEY = 'super-secret-key-12345'`）
- 弱密码验证（只检查长度 >= 6）
- 明文密码比较（`inputPassword === storedPassword`）
- 用 `eval()` 解析用户配置（代码注入风险）
- 登录返回时带上了用户密码字段

**database.js 的问题**：
- 硬编码数据库凭据（`password: 'root123'`）
- SQL 注入（直接拼接 `${username}` 到 SQL）
- 日志里打印完整用户数据（包括密码）

> 💡 这些示例代码的设计本身就很教学——覆盖了最常见的安全漏洞类型，确保审查器「一定有事可做」。

---

### 四、第二步：创建 code-reviewer 子代理

完整配置文件 `.claude/agents/code-reviewer.md`：

```yaml
---
name: code-reviewer
description: Review code changes for quality, security, and best practices. Proactively use this after code modifications.
tools: Read, Grep, Glob, Bash
model: sonnet
---

You are a senior code reviewer...

## Review Dimensions
### Security (Critical Priority)
- SQL injection, XSS, hardcoded secrets, auth issues, input validation

### Performance
- N+1 queries, memory leaks, blocking ops

### Maintainability
- Code complexity, error handling, naming

### Best Practices
- SOLID violations, anti-patterns, code duplication
```

**设计决策解析**：

| 配置项 | 选择 | 理由 |
|--------|------|------|
| `tools` | Read, Grep, Glob, Bash | **没有 Edit 和 Write**——最小权限原则 |
| `model` | sonnet | 审查需要较强分析力，haiku 可能漏问题，opus 太贵 |
| `description` | 含 "Proactively" | 告诉 Claude 可以主动在代码修改后自动调用 |

> 💡 注意 tools 里有 Bash（用于 `git diff` 等只读操作），但没有 Edit/Write。Bash 不等于写权限——它是为了读取变更内容。

---

### 五、第三步：运行与第四步：验证权限边界

**调用方式**：
- 显式：`让 code-reviewer 审查 src/ 目录下的所有代码`
- 自动：`用子代理帮我看看代码有没有安全问题`（Claude 根据 description 自动选择）

**验证权限**——尝试让审查器修复问题：
```
让 code-reviewer 修复 auth.js 中的硬编码密钥问题
→ code-reviewer 只有读取权限，无法修改文件。
```

> 💡 这个验证非常关键——用一次「越权尝试」来确认硬边界确实生效。工程上的安全感不来自「相信它不会做」，来自「确认它做不了」。

---

### 六、从代码审查延伸到影响面分析

黄佳老师引用了一个评论区的真实事故案例：

> 开发者让 AI 改了存量系统的老功能链路，代码没 bug，但上线后用户端 7 秒拿不到结果。原因是 **AI 不知道全链路影响面**——没被告知哪些下游服务会受影响、SLA 是多少。

**核心洞察**：不是 AI 写错了代码，而是 AI **没有被赋予「在设计阶段审视影响面」的职责和上下文**。

由此引出 **impact-analyzer** 子代理：

```yaml
---
name: impact-analyzer
description: Analyze the impact scope of code changes on the full call chain.
tools: Read, Grep, Glob, Bash
model: sonnet
permissionMode: plan
skills:
  - chain-knowledge
  - recent-incidents
---
```

关键设计：
- `permissionMode: plan`——系统级只读保障（即使有 Bash 也无法写入）
- `skills`——启动时自动注入链路拓扑和事故记录，子代理「带着知识开始工作」

---

### 七、SubAgent + Skill 的组合模式

这是全文最有价值的设计模式之一：

```
SubAgent 提供：独立上下文 + 工具权限 + 职责边界
Skill   提供：领域知识（链路拓扑、SLA、历史事故）
        ↓
子代理启动前，Skill 内容通过 skills 字段注入
        ↓
子代理「带着知识」开始分析
```

对比两种做法：

| 做法 | 效果 |
|------|------|
| prompt 里写「请参考 doc/chain.md」 | 子代理需要自己去找文件、读文件、理解内容，不确定它会不会做 |
| `skills: [chain-knowledge]` | 知识在启动时已经注入上下文，子代理一定能用 |

> 💡 这就是正文里反复强调的：**知识注入是系统级的（不依赖 prompt 提示），分析能力是工具级的（受 tools 和 permissionMode 约束）**。

---

### 八、什么时候该创建子代理？决策框架

| 该创建 | 不该创建 |
|--------|---------|
| 任务有明确职责边界 | 一次性任务 |
| 需要权限控制 | 简单 prompt 模板 → 用 Skill |
| 需要上下文隔离 | 自动化触发 → 用 Hook |
| 反复使用的标准化流程 | |
| 多个工具需要组合 | |

黄佳老师自己的例子：训练大模型时，初期日志分析在主对话做。后来日志越来越多，上下文被淹没了，才反推出「需要一个专门的日志分析 SubAgent」。**决策是动态的，不是一开始就知道。**

---

## 核心框架

### 痛点驱动设计五步法

```
工程痛点 → 缺什么能力 → 设计职责边界 → 选工具组合 → 配置子代理
```

### code-reviewer 配置速查

```yaml
name: code-reviewer
tools: Read, Grep, Glob, Bash   # 无 Edit/Write
model: sonnet                     # 平衡能力与成本
# description 要写清：做什么 + 什么时候用 + Proactively
```

### SubAgent + Skill 组合模式

```
skills: [chain-knowledge, recent-incidents]
→ 子代理启动时自动注入 → 带着知识开始分析
```

---

## 技术关键词（待深入）

| 关键词 | 文中位置 | 理解程度 | 后续跟进 |
|--------|---------|---------|---------|
| **impact-analyzer** | 影响面分析子代理设计 | 概念清楚 | 可以在自己项目里试配一个 |
| **chain-knowledge Skill** | 链路拓扑知识的结构化存储 | 理解了，但没实操 | 结合 Skills 章节深入 |
| **permissionMode: plan** | 系统级只读保障 | 理解了，区别于 tools 白名单 | 实操验证 |

---

## 我的疑惑与待验证

1. Bash 的边界在哪——code-reviewer 有 Bash 是为了 `git diff`，但如果 prompt 让它执行 `rm -rf /` 会怎样？Bash 的权限是受 prompt 约束还是受系统约束？
2. `skills` 字段注入 Skill 内容——这是全文注入还是渐进式加载？如果 Skill 很大（比如几千行的链路文档），会影响子代理的上下文空间吗？
3. impact-analyzer 依赖的 chain-knowledge Skill 需要人工维护——链路变了谁来更新？有没有自动化方案？

---

## 相关链接

- 📁 [原文原始数据](../article-origin/006/)
- 📦 [课程 GitHub - 03-SubAgents 实战项目](https://github.com/huangjia2019/claude-code-engingeering)
