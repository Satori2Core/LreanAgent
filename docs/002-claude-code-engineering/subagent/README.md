# SubAgent 专题

> Claude Code 的多代理协作能力——从单 Agent 的安全分工到多 Agent 的团队协作。
>
> 覆盖 004-010 共七篇笔记。

---

## 核心结论

**SubAgent 解决的不是「能力问题」，而是「工程风险」**——上下文污染、权限越界、角色混乱、锚定偏见、经验散落。

两层模型：`Sub-Agents（结构化分工）→ Agent Teams（协作型认知）`

---

## 笔记索引

| 序号 | 笔记 | 核心内容 |
|------|------|---------|
| 004 | [Sub-Agents 核心概念与应用价值](../004-SubAgents核心概念/) | 什么是子代理？隔离/约束/复用三大价值。配置文件（frontmatter + prompt）详解。内置 Explore/Plan/General-purpose。 |
| 005 | [从 Sub-Agents 到 Multi-Agent 工程指南](../005-Multi-Agent工程指南/) | 四种设计模式：Sub-Agent / Skills / Handoffs / Router。LangChain 量化对比。模式选择速查表。 |
| 006 | [只读型和安全型子代理实战](../006-只读型子代理实战/) | 从零创建 code-reviewer。痛点驱动设计五步法。SubAgent + Skill 组合：impact-analyzer。 |
| 007 | [高噪声任务处理：测试运行器与日志分析器](../007-高噪声子代理实战/) | 信噪比决策框架。test-runner（haiku）+ log-analyzer（sonnet）。三层输出格式设计法 + 四原则。 |
| 008 | [并行探索与流水线编排](../008-并行探索与流水线编排/) | 并行（独立任务）+ 流水线（依赖任务）+ 混合模式。交接契约。主对话四种编排姿态。 |
| 009 | [Agent Teams 会话协作架构](../009-Agent-Teams/) | Teammates 互相通信。四大模式：竞争假设/分层评审/模块化开发/规划-审批。选型决策树。 |
| 010 | [SubAgent 专题阶段性总结](../010-SubAgent阶段总结/) | 七讲知识体系整合。完整选型决策树。五大工程原则。支付超时排查五阶段完整案例。 |

---

## 选型速查

```
需要多个 workers？
├── 否 → 单 Agent / 主对话
└── 是 → 需要互相通信？
    ├── 否 → Sub-Agent（独立→并行 / 依赖→流水线）
    └── 是 → Agent Teams（竞争假设 / 分层评审 / 模块开发 / 规划审批）
```

---

## 相关实践

- [article-fetcher agent](../../../.claude/agents/article-fetcher.md) — 文章拉取子代理（deepseek-chat）
- [comment-filter agent](../../../.claude/agents/comment-filter.md) — 评论筛选子代理（deepseek-chat）
