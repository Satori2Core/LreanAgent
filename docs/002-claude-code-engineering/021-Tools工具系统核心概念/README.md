# 021 · 18｜庖丁解牛：Tools 工具系统核心概念

> 📖 原文出处：[极客时间 - 黄佳《Claude Code 工程化实战》](https://time.geekbang.org/column/article/957210)
>
> 📅 学习时间：2026-07-13

---

## 这篇文章在回答什么问题？

1. **Claude Code 的内置工具是怎么组织的？** 十几个原生工具，为什么偏偏是这些，不是更多也不是更少？核心设计哲学：五个原子操作覆盖一切软件工程任务。
2. **Agentic Loop 是什么？** Claude（推理）+ Tools（行动）+ Harness（连接两者）= Agent 的工作循环。Skills、Hooks、MCP 都在 Harness 上工作。
3. **工具的风险等级怎么分？** 只读不可逆（Read/Grep/Glob/WebSearch）→ 只读可逆（WebFetch）→ 写入可逆（Edit/Write）→ 执行不可逆（Bash）。

---

## 原文转述

### 一、Agentic Loop：工具驱动智能循环

没有工具时，Claude 只能输出文本——能思考、分析、给建议，但不能行动。有了工具后：

```
用户: 修复 src/api.js 中的 bug
Claude: 思考 → 调用 Read 工具获取文件 → 发现第 42 行问题
        → 调用 Edit 工具修改 → 调用 Bash 运行测试 → 确认修复
```

**Claude Code 的三层架构**：

```
Claude 模型（推理）→ Agentic Harness（连接层）→ Tools（行动）
                                   ↑
                    Skills / Hooks / MCP 都在这一层工作
```

> 💡 你配置的所有扩展不是直接修改 Claude 模型本身，而是在 Harness 上工作。Skills 改变「怎么做」、Hooks 拦截「能不能做」、MCP 扩展「能做什么」。

---

### 二、内置工具六分类

| 类别 | 工具 | 用途 |
|------|------|------|
| 文件操作 | Read/Write/Edit/MultiEdit | 与代码交互 |
| 搜索 | Glob/Grep | 「眼睛」——先看清再动手 |
| 执行 | Bash | 最强也最危险 |
| 网络 | WebFetch/WebSearch | 突破本地边界 |
| 编排 | Task/Agent/TodoWrite | 管理子代理和任务流程 |
| 辅助 | backgroundTask 等 | 后台任务管理 |

---

### 三、工具风险等级

```
只读不可逆：Read Grep Glob WebSearch     → 无需确认
只读可逆：  WebFetch                     → 可能写缓存
写入可逆：  Write Edit MultiEdit         → 改代码需确认
执行不可逆：Bash                         → 始终确认
```

> 💡 这就是子代理设计中为什么 `tools: Read, Grep, Glob` 只会读不会改——低风险工具有利于构建无人值守的自动化流程。

---

### 四、工具设计哲学：五个原子操作

Claude Code 不提供「重构工具」「调试工具」「部署工具」——而是提供**原子原语**：

```
感知（Read）→ 搜索（Grep/Glob）→ 修改（Write/Edit）→ 执行（Bash）→ 网络（WebFetch）
```

任何软件工程任务都可以分解为这五个原子操作的组合。就像编程语言的少量关键字能表达一切程序，工具的少量原语能覆盖一切开发任务。复杂能力从原语组合中**涌现**出来。

---

## 核心框架

### 三层架构

```
Claude(推理) → Harness(连接层，Skills/Hooks/MCP) → Tools(行动)
```

### 风险等级

```
只读不可逆 < 只读可逆 < 写入可逆 < 执行不可逆
```

---

## 相关链接

- 📁 [原文原始数据](../article-origin/021/)
- 📖 [Claude Code 工具参考](https://code.claude.com/docs/en/tools-reference)
- 🗂 [阅读指南](../000-阅读指南/)
