# 004 · 03｜分而治之：Sub-Agents 的核心概念与应用价值

> 📖 原文出处：[极客时间 - 黄佳《Claude Code 工程化实战》](https://time.geekbang.org/column/article/943368)
>
> 📅 学习时间：2026-06-30

---

## 这篇文章在回答什么问题？

1. **Claude Code 越用越「健忘」是怎么回事？** 不是模型退化了，是对话上下文被一次次「中间过程」污染了。这个问题怎么从工程上解决？
2. **Sub-Agent 到底是什么？** 前面两篇都在提 Sub-Agent，这一篇终于正式讲清楚：「把一个大脑拆成多个岗位角色」，每个岗位有独立的上下文窗口和权限边界。
3. **什么时候该用 Sub-Agent，什么时候不该用？** 不是所有任务都值得派出子代理。有四类场景特别适合，也有几类场景不适合。
4. **Sub-Agent 配置文件怎么写？** 哪些字段必填、哪些可选？`description` 怎么写才能让 AI 知道什么时候自动调用？

---

## 原文转述

### 一、问题起源：上下文污染

黄佳老师用一个经典的崩溃场景开场：

```
你让 Claude 跑测试 → 500 行日志
又让它分析代码结构 → 200 行 grep
再让它改个 bug → 对话上下文已经被中间过程塞爆
真正重要的信息淹没在噪声里
```

**关键洞察**：这些中间输出有一个共同特征——它们对「当下执行」是必要的，但对「后续决策」是噪声。而 Claude Code 的主对话是线性追加的，不会自动过期，「把临时执行数据当成了长期决策记忆」。

💡 打个比方：主对话像你的办公桌。每次任务产生的中间产物（测试日志、搜索输出、临时分析）就像草稿纸，全堆在桌上。桌面上有用的文件（决策、结论）早晚会被草稿纸淹没。Sub-Agent 就像一个分拣员——把草稿纸丢到碎纸机，只把结论放你桌上。

---

### 二、Sub-Agent 的本质

**一句话定义**：Sub-Agent 是一个「专职小助手」——带着自己的规则、工具权限、独立上下文窗口去完成某一类任务，只把结果摘要带回来。

核心对比：

| | 主对话（老板） | Sub-Agent（员工） |
|------|-----------|----------|
| **职责** | 理解意图、编排任务、做决策 | 执行具体任务、产出结果 |
| **上下文** | 长期决策记忆 | 独立上下文，执行完即释放 |
| **权限** | 全部工具 | 被精确控制的工具子集 |

关于「独立上下文窗口」这句特别重要：**不是因为它「聪明」或「更强」，而是因为它是 Claude Code 里唯一一个结构上允许「执行完即丢弃」的东西。**

💡 这个「执行完即丢弃」是 Sub-Agent 最重要的设计特征。Skills 的执行过程留在主对话里，Commands 的执行过程也留在主对话里，只有 Sub-Agent 干完活自动清场。

---

### 三、三大核心价值：隔离、约束、复用

#### 隔离 —— 解决上下文污染

这是 Sub-Agent 最重要的价值。主对话只看到结论，不需要承载 500 行的搜索过程。

```
主对话上下文                   子代理上下文（独立，执行完释放）
┌─────────────────────┐      ┌─────────────────────────┐
│ 用户：帮我分析 bug   │      │ 任务：查 bug 相关文件    │
│ Claude：让我看看...  │ ←返回 │ [500 行搜索日志...]     │
│ Claude：发现 3 个文件│      │ 结论：3 个相关文件       │
└─────────────────────┘      └─────────────────────────┘
```

#### 约束 —— 解决行为不可控

不是靠提示词「请你不要改」，而是靠工具权限「你物理上改不了」。

```yaml
# 只读型—代码审查
tools: Read, Grep, Glob

# 开发型—bug 修复
tools: Read, Write, Edit, Bash

# 研究型—技术调研
tools: Read, WebFetch, WebSearch
```

💡 这解决了 AI 工程中一个核心矛盾：你让 Claude 审查代码，它可能顺手帮你「修」了它觉得有问题的地方。有了 Sub-Agent 的权限边界，好意的越权也不会发生。

#### 复用 —— 解决经验无法沉淀

配置保存为文件 → 进版本控制 → 团队共享 → 跨项目复用 → 持续迭代优化。

```
.claude/agents/
├── test-runner.md
├── code-reviewer.md
├── log-analyzer.md
└── bug-fixer.md
```

💡 这三点合在一起，标志着 Claude Code 从「对话技巧」正式跨入「工程系统」。

---

### 四、内置 Sub-Agent

你可能已经在用了，只是没意识到。Claude Code 内置了三类：

| 内置代理 | 职责 | 特点 |
|---------|------|------|
| **Explore** | 翻项目、找位置 | 快速只读，三档模式（quick/medium/very thorough） |
| **Plan** | 动手前先想清楚 | 收集上下文、梳理依赖、生成实施路径 |
| **General-purpose** | 全能型员工 | 能探索、能修改、能推进，完整工具集 |

💡 Explore 是最常用的——当你问「这个项目的认证逻辑在哪」，CC 会派出 Explore 而不是在主对话里一行行 grep。

---

### 五、什么时候该用 / 不该用

**该用的四类场景**：

| 场景 | 特征 | 示例 |
|------|------|------|
| 高噪声输出 | 执行过程产生大量中间信息，主对话只关心结论 | 跑测试、扫日志、代码搜索 |
| 明确角色边界 | 只想让 AI「看」不想让它「动手」 | 代码审查、安全审计 |
| 可并行研究 | 多个独立探索可以同时进行 | 同时调研认证/数据库/API |
| 流水线式任务 | 可拆成清晰阶段的串行任务 | 定位→审查→修复→验证 |

**不适合的场景**：
- 需要频繁来回确认需求、不断调整方向
- 任务各阶段高度耦合，每步强依赖上一步的详细过程
- 非常简单的小任务（启动 Sub-Agent 本身有开销）

---

### 六、一条关键约束：Sub-Agent 不能嵌套

**Sub-Agent 不能再调用 Sub-Agent。** code-reviewer 不能在执行中再派出 security-scanner。

这意味着：
- **所有编排必须由主对话完成**——如果你需要「先审查再修复」，主对话依次调用两个
- **主对话是唯一的调度中心**
- **Sub-Agent 里需要用 Skills 来复用知识**，而不是再嵌套一个 Sub-Agent

---

### 七、配置文件详解

格式：**Markdown + YAML frontmatter**。

```yaml
---
name: code-reviewer
description: Review code for quality, security, best practices. Use proactively after code changes.
tools: Read, Grep, Glob
model: sonnet
permissionMode: plan
skills:
  - chain-knowledge
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/validate-readonly-query.sh"
---

你是一个代码审查专家...
```

#### 关键字段说明

| 字段 | 必填 | 说明 |
|------|------|------|
| `name` | ✅ | 角色标识 |
| `description` | ✅ | **最重要**——决定 Claude 何时自动调用。要写清楚「做什么」+「什么时候用」 |
| `tools` | 可选 | 白名单——「只能用这些」 |
| `disallowedTools` | 可选 | 黑名单——「除了这些都能用」。**不要和 tools 同时用** |
| `model` | 可选 | sonnet/haiku/opus，在性能与成本间权衡 |
| `permissionMode` | 可选 | `plan`=系统级只读保障，比 prompt 约束更可靠 |
| `skills` | 可选 | 子代理启动时预加载的 Skill 内容（不会自动继承主对话的 Skills） |
| `hooks` | 可选 | 只在该子代理运行期间生效，结束后自动清理 |

#### description 是设计的核心

```yaml
# ❌ 太模糊——Claude 不知道什么时候该用它
description: A code reviewer

# ✅ 说明做什么 + 什么时候用 + 关键词 Proactively
description: Review code changes for quality, security vulnerabilities, and best practices. Use proactively after code is modified or when user asks for code review.
```

#### 典型工具组合

```
只读型（审计）         研究型（搜集）         开发型（读写改）
Read                   Read                  Read
Grep                   Grep                  Write
Glob                   Glob                  Edit
                       WebFetch              Bash
                       WebSearch             Glob
                                             Grep
```

#### 存储位置与优先级

```
用户级：~/.claude/agents/        ← 所有项目可用，通用角色
项目级：项目/.claude/agents/      ← 仅当前项目，优先级更高
```

> 💡 同名时项目级覆盖用户级。

---

### 八、三种创建方式

| 方式 | 操作 | 适合 |
|------|------|------|
| **交互式** | 输入 `/agents` → 选 Create new agent → 向导引导 | 新手 |
| **手写配置** | 直接创建 `.claude/agents/xxx.md` | 精确控制、版本管理 |
| **CLI 临时** | `--agents` 参数传入 JSON，会话结束后消失 | CI/CD |

---

### 九、运行模式

- **前台/后台**：Claude 自动选择，也可以手动说「run this in the background」
- **恢复（Resume）**：子代理完成后可以用 `agent ID` 恢复它继续工作——保留完整对话历史，从上次停下的地方继续
- **权限预请求**：启动前 Claude 会预先请求可能需要的所有权限（因为后台无法弹窗确认）

---

## 核心框架

### Sub-Agent 四大工程价值

| 价值 | 解决什么问题 | 对应软工命题 |
|------|------------|-----------|
| **隔离** | 上下文污染（执行噪声混入决策记忆） | 内存管理 |
| **约束** | 行为不可控（提醒靠自觉 → 系统硬边界） | 安全边界 |
| **复用** | 经验不可沉淀（一次性对话 → 可版本化资产） | 组织效率 |
| **并行** | 串行瓶颈 | 并发加速 |

### 什么时候用 / 不用

| 该用 | 不该用 |
|------|--------|
| 高噪声输出任务 | 频繁来回确认需求 |
| 明确角色边界 | 高度耦合的任务 |
| 可并行研究 | 非常简单的小任务 |
| 可拆分流水线 | |

---

## 技术关键词（待深入）

| 关键词 | 文中怎么说的 | 理解程度 | 后续跟进 |
|--------|-------------|---------|---------|
| **permissionMode: plan** | 系统级只读保障，比 prompt 约束更可靠 | 大概理解，没实操 | 后续实战验证 |
| **skills 字段** | 子代理启动时预加载 Skill 内容，不会自动继承主对话 Skills | 知道了，需要实验 | 结合 Skills 章节 |
| **hooks 字段** | 子代理专属生命周期的钩子，结束后自动清理 | 概念清楚 | Hook 专题 |
| **agent ID / resume** | 恢复之前的子代理，保留完整对话历史 | 新概念，有意思 | 实操试试 |

---

## 我的疑惑与待验证

1. Sub-Agent 的独立上下文窗口大小是多少？和主对话共享 token 配额还是独立计算？
2. `permissionMode: plan` 和 `tools: Read, Grep, Glob` 有什么区别？如果不给 Bash，是不是就等于只读了？
3. description 的触发机制——是关键词匹配还是语义理解？写「Proactively」真的有用吗？
4. 这篇没提到 Sub-Agent 和后续 MCP、A2A 的关系——它们之间的边界在哪？

---

## 评论区高价值讨论

（评论区走的是原文 highlight 列表，暂不深入——等后续实操篇的评论区讨论更实用。）

---

## 相关链接

- 📁 [原文原始数据](../article-origin/004/)
- 📦 [课程 GitHub - 03-SubAgents 示例](https://github.com/huangjia2019/claude-code-engingeering)
- 🔗 [OpenClaw](https://github.com/openclaw/openclaw)
