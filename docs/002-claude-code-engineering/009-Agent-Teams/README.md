# 009 · 08｜群策群力：Agent Teams 会话协作架构

> 📖 原文出处：[极客时间 - 黄佳《Claude Code 工程化实战》](https://time.geekbang.org/column/article/945358)
>
> 📅 学习时间：2026-07-02

---

## 这篇文章在回答什么问题？

1. **Sub-Agent 做不到什么？** 子代理只能向主对话汇报，不能互相通信。遇到需要「互相讨论、挑战彼此结论」的场景怎么办？
2. **Agent Teams 是什么？** Claude Code 新出的实验性功能——多个 Claude Code 实例作为团队协作，Teammates 可以互发消息、共享发现。
3. **Agent Teams 有哪些协作模式？** 竞争假设（辩论）、分层评审（各审各维度）、模块化开发（分工写代码）、规划-审批（先交方案再干）。
4. **什么时候用 Sub-Agent、什么时候用 Agent Teams？** 核心判断：Workers 需要互相通信吗？不需要→Sub-Agent，需要→Agent Teams。

---

## 原文转述

### 一、从「任务委托」到「团队协作」

黄佳老师用一个 bug 调查场景引出核心差异：

```
系统出现奇怪 bug——用户登录后偶尔会话丢失，怀疑三个方向：
假设 A：JWT token 过期时间有问题
假设 B：Redis session 的竞态条件
假设 C：负载均衡器 sticky session 配置
```

**用子代理**：三个子代理各自独立调查，只向主对话汇报。子代理 B 看不到子代理 C 的发现，可能错过了关联线索。

**用 Agent Teams**：如果子代理 B 能看到子代理 C 的发现，它可能会说「等等，Redis 连接数上限可能是因为 sticky session 5 分钟后切换了服务器，导致新的 Redis 连接被创建」。

这就是 Agent Teams 的核心价值：**让代理之间能够直接交流、互相挑战、协作推进。**

**架构对比**：

| | Sub-Agent | Agent Teams |
|------|---------|------------|
| **通信** | 只向主对话汇报 | Teammates 互相发消息 |
| **关系** | 老板→员工 | 团队成员 |
| **适用** | 独立子任务 | 需要讨论和跨视角发现 |

---

### 二、Agent Teams 怎么工作

**启用**（实验性功能，默认关闭）：

```json
{ "env": { "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1" } }
```

**组件**：Lead（协调者）+ Teammates（执行者）+ Task List（共享任务）+ Mailbox（消息系统）

**Team 生命周期**：创建团队 → Lead 分配任务 → Teammates 各自工作 + 互相通信 → Lead 综合结果 → 清理团队

> 💡 Teams 是一次性的「函数模型」——调用即生、返回即死，不是常驻进程。评论区作者解释了为什么：常驻进程上下文不断膨胀、历史偏见累积、token 持续消耗。函数模型 + 文件持久化是更务实的方案。

---

### 三、实战：全栈 Bug 猎人

课程项目模拟了一个 Express.js 电商应用，刻意植入 4 个级联 bug：

```
Bug 1: DB 连接池太小 → 连接耗尽
Bug 2: Redis Session 不处理重连 → Session 写入静默失败
Bug 3: 订单查询 N+1 → 大量连接被占用 → 加剧 Bug 1
Bug 4: 缓存 key 缺少用户标识 → 数据泄漏
```

启动 4 个侦探 Teammate：Session 侦探、数据库侦探、缓存侦探、架构侦探。每个持有不同假设，互相分享发现、挑战结论，最终拼出完整级联故障链。

**为什么子代理搞不定这个场景？** 单独调查者容易「锚定」——找到一个 bug 就满意了不继续挖。bug 之间的关联需要跨视角发现。只有侦探们互相通信，才能看到级联效应。

---

### 四、四大协作模式

**模式一：竞争假设**——多假设并行验证，互相辩论推翻

```
生成 5 个 agent teammates 调查不同的假设。
让它们互相对话，试图推翻对方的理论，像科学辩论一样。
```

💡 工程价值：避免锚定效应。多个独立调查者互相反驳，幸存的理论更可能是真正根因。

**模式二：分层评审**——多维度并行审查（安全/性能/测试）

**模式三：模块化开发**——每人负责不同模块，通过共享任务列表协调。关键机制：任务依赖声明 + 自动解锁 + 文件所有权。

**模式四：规划-审批**——Teammate 先提交计划，Lead 审批通过后才能执行。适合高风险任务。

---

### 五、选型决策：Sub-Agent vs Agent Teams

```
Workers 需要互相通信吗？
├── 否 → Sub-Agents（低 token、简单协调、并行探索/流水线）
└── 是 → Agent Teams（支持讨论挑战、共享任务、协作开发/多角度审查）
```

一句话：**只需汇报结果 → Sub-Agent；需要互相讨论 → Agent Teams。**

Token 成本：Agent Teams > Sub-Agents > 单会话。每个 Teammate 是独立实例，消息通信消耗额外 token。

---

## 核心框架

### 四大模式速查

| 模式 | 适合场景 | 核心机制 |
|------|---------|---------|
| **竞争假设** | 根因不明，多方向验证 | 辩论机制，存活者可能是真根因 |
| **分层评审** | PR Review，多维度评估 | 并行审查不同维度 |
| **模块化开发** | 新功能，多模块协作 | 任务依赖 + 文件所有权 |
| **规划-审批** | 高风险任务 | 先交方案，审批后执行 |

### Sub-Agent vs Agent Teams

| 维度 | Sub-Agent | Agent Teams |
|------|---------|------------|
| 通信 | 只向主对话 | Teammates 互发消息 |
| Token | 较低 | 显著更高 |
| 协调 | 主对话单点 | Lead + Mailbox |
| 适合 | 独立任务 | 需要讨论/挑战 |

---

## 评论区高价值讨论

### 🔥 1. Agent Teams 能常驻吗？

**读者 追风少年（9 赞）**：Agent Team 用完就销毁，能不能设计一个常驻的 Team？

**作者答**：Claude Code 选择了「函数模型」——调用即生、返回即死。不是技术限制，是刻意设计。常驻进程上下文膨胀、历史偏见累积、token 持续消耗。函数模型 + 文件持久化更务实：**把值得记住的写成文件，不值得记住的让它消失。** Skill 就是 checklist，history/ 就是工作日志。

> 💡 这跟人类专家的工作方式一样——你不会记得每次代码审查的细节，但会把反复出现的问题写进团队 checklist。

---

### 🔥 2. 国内模型跑 Agent Teams 报错

**读者 黄琨（2 赞）**：换成国内模型后，除了领队 agent，其他 agent 都报 `API Error: 400 Not supported model claude-opus-4-6`。

**作者答**：感谢反馈，可以 report 这个 issue 给 CC repo。

> 💡 这条跟我们的 DeepSeek 场景直接相关——Agent Teams 可能对非 Claude 模型兼容性不够好，毕竟是实验性功能。

---

### 🔥 3. Boris Cherny 说：大多数情况下不要手动创建 Agent

**读者 Geek_e153bf（1 赞）**：Claude Code 团队成员 Boris Cherny 现场分享时表示，大多数情况下不喜欢也不建议手动创建 agent——让 Claude 根据任务自动创建即可。

**作者答**：完全同意。实践中，看场景，不一定需要创建 Agent（CC 能自动创建子 agent）。未来可能是 Harness 根据情况自动创建并清理 Agent。

---

### 🔥 4. 其他碎片

- **Agent Teams 不替代 BMAD/spec-kit**（南方，0赞）：Teams 在「执行能力」层面强，但「方法论」层面是空白的——不告诉你该做什么阶段、每阶段产出什么制品。
- **Teams 之间不通信？**（勇敢的大白，0赞）：有读者反馈实际运行时 teammates 之间无交互——确认是没有开启实验性功能开关。
- **跨 repo 支持**（TIAN，1赞）：目前不支持 per-teammate cwd，所有 Teammate 继承 Lead 工作目录。

---

## 相关链接

- 📁 [原文原始数据](../article-origin/009/)
- 📖 [Claude Code Agent Teams 官方文档](https://code.claude.com/docs/en/agent-teams)
- 📦 [课程 GitHub - 03-SubAgents 实战](https://github.com/huangjia2019/claude-code-engingeering)
