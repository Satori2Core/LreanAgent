# 019 · 16｜未雨绸缪：Hooks 高级模式与工程实践

> 📖 原文出处：[极客时间 - 黄佳《Claude Code 工程化实战》](https://time.geekbang.org/column/article/954158)
>
> 📅 学习时间：2026-07-12

---

## 这篇文章在回答什么问题？

1. **Claude 说「做完了」，但谁来验收质量？** PreToolUse 管入口、PostToolUse 管过程——但缺少终检。Stop Hook 填补了这个空白：任务完成后自动运行质量门控，不通过就不让停。
2. **Stop Hook 怎么防止死循环？** `continue: true` 让 Claude 继续修复，但一直修不好怎么办？用 `stop_hook_active` 字段设重试上限。
3. **怎么在子代理启动/停止时自动介入？** SubagentStart（注入上下文）+ SubagentStop（验证交付质量）把 Hooks 和 SubAgents 两大机制连接起来。

---

## 原文转述

### 一、Stop Hook——任务完成时的质量门控

黄佳老师用工厂流水线类比：

```
PreToolUse  = 入口安检（原料合格吗？）
PostToolUse = 过程质检（工序产出对吗？）
Stop Hook   = 出厂验收（成品整体达标吗？）← 补上第三环
```

Stop Hook 区别于其他 Hook 的核心能力：**让 Claude 继续工作**。

```json
{"decision": "block", "reason": "Tests failing, please fix", "continue": true}
```

`continue: true` 创造一个自动循环：Claude 认为完成 → Stop Hook 检查 → 测试失败 → 反馈失败信息给 Claude → Claude 继续修复 → 再次完成 → 再次检查 → **直到所有检查通过才真停**。

> 💡 这从根本上改变了质量保证的模式：不是「做完了再检查」，而是「检查通过了才算做完」。

**实战：自动测试门控**。Stop 时运行测试而不是每次文件修改后——因为中间状态测试必然失败（改了接口没改实现）。只在 Claude 认为「全部完成」时测试才有意义。

脚本自动检测项目类型（Node.js/Python/Go/Rust），测试失败截断前 50 行返回给 Claude（信噪比思维），没有测试脚本的项目不报错直接放行。

**防止死循环**：用 `stop_hook_active` 字段。当 Claude 因 Stop Hook 继续工作后，下一次 Stop 时该字段为 `true`。脚本检查此字段来控制重试上限：

```bash
if [ "$(echo "$INPUT" | jq -r '.stop_hook_active')" = "true" ]; then
    exit 0  # 已经重试过了，这次放行
fi
```

---

### 二、SubagentStart 与 SubagentStop

**SubagentStart**：子代理启动时触发，不能阻止启动，但可以通过 `additionalContext` 注入上下文。

```json
{"additionalContext": "Team standards: camelCase, max 100 chars, JSDoc for public APIs"}
```

价值在于**自动化上下文注入**——不需要每次调用子代理时手动提醒，不需要把规范写到子代理 prompt 里（占用上下文）。

**SubagentStop**：子代理完成时触发，可以和 Stop 一样用 `decision: "block"` 阻止完成。独特字段 `agent_transcript_path`——可以读取子代理完整对话历史来判断质量。

**实战：验证代码审查完整性**。好审查不仅要发现问题，还要提供解决方案。脚本用关键词匹配检查审查报告是否同时包含 issue 和 suggestion——有 issue 没 suggestion → block。

---

### 三、完整 Hook 系统设计

单个 Hook 解决单个问题，但真正的工程价值在**多 Hook 组合成系统**。

**安全钩子系统**：阻止危险命令 + 保护敏感文件 + 记录审计日志。三层防线各司其职，防御不再依赖任何一层的「不遗漏」。

**质量钩子系统**：自动格式化 + 自动测试门控 + Prompt 代码审查。逐层把关，确保交付物质量。

---

### 四、三维决策框架

```
维度 1: 什么时候触发？
  PreToolUse（阻止坏人） → PostToolUse（自动善后） → Stop（质量门控）
  SubagentStart（注入上下文） → SubagentStop（验证交付）

维度 2: 用什么执行？
  Command（确定规则） → Prompt（LLM 判断） → Agent（深度验证）

维度 3: 怎么做响应？
  allow → ask → deny / block + continue
```

---

## 核心框架

### 三层防线全景

```
PreToolUse  → 入口安检（阻止危险命令、保护敏感文件）
PostToolUse → 过程质检（自动格式化、Lint 检查）
Stop        → 出厂验收（自动测试、完整性检查）
```

### 子代理事件

```
SubagentStart → 注入上下文（cannot block）
SubagentStop  → 验证交付（can block + continue）
```

### 防止死循环

```bash
stop_hook_active == true → 放行（不让 Claude 无限重试）
```

---

## 相关链接

- 📁 [原文原始数据](../article-origin/019/)
- 📝 [上一讲：Hooks 事件驱动自动化](../018-Hooks事件驱动自动化/)
- 🗂 [阅读指南](../000-阅读指南/)
