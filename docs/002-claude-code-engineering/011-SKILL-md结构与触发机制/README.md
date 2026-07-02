# 011 · 09｜触类旁通：SKILL.md 结构与触发机制

> 📖 原文出处：[极客时间 - 黄佳《Claude Code 工程化实战》](https://time.geekbang.org/column/article/945937)
>
> 📅 学习时间：2026-07-02

---

## 这篇文章在回答什么问题？

1. **Skills 到底是什么？** 不是工具、不是子代理、不是 Hook——它回答的是「怎么做，以及何时做」这个独特的问题。它是一种「可操作知识结构」。
2. **Skills 怎么被触发？** 靠 `description` 字段的语义匹配，不是关键词匹配。Claude 读取所有 Skill 的 description，通过语义理解判断当前对话是否需要加载某个 Skill。
3. **渐进式加载省多少 Token？** 所有 description 常驻上下文（约占 3-5%），Skill 全文仅在触发时加载。节省比例高达 78%~98%。
4. **怎么写一个好的 description？** 公式：`[做什么] + [怎么做] + [什么时候用]`
5. **参考型 vs 任务型 Skill 怎么区分？** 参考型让 Claude 自动判断加载（如 API 规范），任务型通常手动触发且配合 `disable-model-invocation: true`。

---

## 原文转述

### 一、Skills 出现的背景：知识太多，全塞进上下文不现实

黄佳老师用一个团队场景开头：代码风格指南十几页、Git 提交规范三四种、API 设计有版本约定、安全审查有检查清单……这些规则不复杂，但数量一多，不可能长期驻留在脑中。

人类工程师的做法是「需要时再查阅」。但如果你把所有规范都塞进 CLAUDE.md，每次对话都在为「可能用不到的知识」支付上下文成本——消耗的不只是 Token，更是模型的注意力。

> 💡 Skills 解决的就是这个问题：与其把所有知识常驻上下文，不如**封装成可独立触发的能力单元**，需要时再加载。

---

### 二、Skills 的准确定义

**Skills 是一种可被语义触发的能力包，包含领域知识、执行步骤、输出规范与约束条件，并在需要时渐进式加载到主 Agent 的认知空间中。**

在 Agent 生态四大支柱中的位置：

| 支柱 | 回答的问题 | 类比 |
|------|----------|------|
| **Tools** | 能做什么 | 人的双手 |
| **SubAgents** | 谁来做 | 团队同事 |
| **Hooks** | 什么时候检查 | 质检流程 |
| **Skills** | 怎么做、何时做 | 标准操作程序(SOP) |

> 💡 Skills 的独特之处：它连接「行动能力」和「语义世界」。Tools 给你函数，Skills 告诉你——在这个项目的世界里，函数应该怎么用、有什么规矩。

---

### 三、Skills = 组织的 SOP 体系（企业本体论视角）

这是全文最有深度的部分。黄佳老师把 Claude Code 技术栈映射到企业组织：

- Tools = 员工的操作工具
- SubAgents = 岗位分工
- Hooks = 质检流程
- CLAUDE.md = 企业文化与通用规章
- **Skills = 标准操作程序（SOP）**

一个成熟企业不会要求员工背诵全部操作手册，而是在具体任务发生时按需查阅 SOP。新员工做代码审查时，参考《代码审查 SOP》，按步骤检查，输出符合模板的报告。Claude 加载 code-review Skill 时做的事情完全一样。

> 💡 **Skills 的真正意义**：组织的「做事方式」第一次在 Agent 系统中获得了结构化存在的形式。经验不再依附于老员工的记忆，而变成模型可以理解、选择和继承的结构。

---

### 四、渐进式加载的量化效果

假设你有 5 个 Skills，每个 SKILL.md 约 1000 tokens：

- 全部常驻：5 × 1000 = 5000 tokens
- 渐进式加载：5 个 description（~200 tokens）+ 1 个完整 Skill（~1000 tokens）= 1200 tokens
- **节省 78%~98%**

这跟前面学过的 SubAgent 上下文隔离有异曲同工之处——都是「不把不需要的信息塞进上下文」。

---

### 五、两种触发方式

```
方式一：用户手动 /skill-name     → 100% 确定触发
方式二：Claude 语义自动判断       → 概率性触发
```

**description 是 Skill 的灵魂**——它不是给人看的文档，而是给 Claude 看的触发器。Claude 选择是否激活一个 Skill，完全依赖语义理解 description。

如果想禁止 Claude 自动触发某个 Skill（比如危险操作），用 `disable-model-invocation: true`——此时 description 甚至不会加载到上下文，Claude 完全看不见它，只有用户手动 `/name` 才能触发。

---

### 六、两种类型：参考型 vs 任务型

| | 参考型 Skill | 任务型 Skill |
|------|-----------|----------|
| **作用** | 影响「怎么做」 | 决定「做什么」 |
| **触发** | Claude 自动判断 | 通常手动触发 |
| **内容** | 规范、约定、标准 | 操作步骤、执行流程 |
| **示例** | API 设计规范 | `/deploy` 部署命令 |
| **disable-model-invocation** | 不设 | 通常设为 true |
| **企业类比** | 行为规范层 | 操作流程层 |

💡 用企业本体论的视角：参考型 Skill 定义「在这个世界里，什么是正确的做法」——它是世界规则。任务型 Skill 定义一次明确的行动——它是世界事件。

---

### 七、description 写作公式

```
description = [做什么] + [怎么做] + [什么时候用]
```

**对比示例**：

```yaml
# ❌ 太模糊
description: Handles PDFs

# ✅ 具体可用
description: Extract text and tables from PDF files, fill forms, merge documents.
Use when working with PDF files or when the user mentions PDFs, forms, or document extraction.
```

**多 Skill 时避免冲突**：

```yaml
# ❌ 两个 description 几乎一样，Claude 分不清
name: unit-testing
description: Write tests for code

name: integration-testing
description: Write tests for code

# ✅ 明确区分
name: unit-testing
description: Write unit tests for individual functions. Use for testing single
functions in isolation, mocking dependencies.

name: integration-testing
description: Write integration tests for system components. Use when testing
how multiple components work together, testing API endpoints end-to-end.
```

---

### 八、Frontmatter 字段速查

```yaml
---
name: my-skill-name              # Skill 标识符（省略则用目录名）
description: What this does      # ★ 最重要：触发器
argument-hint: "[issue-number]"  # 自动补全参数提示
disable-model-invocation: true   # 禁止 Claude 自动触发
user-invocable: false            # 对用户隐藏 /skill-name
allowed-tools:                   # 限制可用工具
  - Read
  - Grep
  - Bash(git:*)                  # 支持 Bash 子命令限制
context: fork                    # 在子代理中隔离执行
agent: Explore                   # fork 时使用的代理类型
model: sonnet                    # 指定执行模型
hooks:                           # Skill 级别的生命周期 Hook
---
```

**关键字段说明**：

- `description`：所有 Skill 的 description 总和有 15,000 字符预算上限，可通过 `/context` 查看
- `allowed-tools`：Skill 激活时列出的工具**无需逐次确认**
- `context: fork` + `agent: Explore`：让 Skill 在子代理的隔离上下文中运行
- `model`：可以给不同 Skill 配不同模型——省钱

---

### 九、参考型 Skill 实战：api-conventions

目录结构：`.claude/skills/api-conventions/SKILL.md`

```yaml
---
name: api-conventions
description: API design patterns. Use when writing or reviewing API endpoints.
allowed-tools: Read, Grep, Glob
---
```

正文内容：URL 命名规范、响应格式、HTTP 状态码、认证方式、版本控制。

**为什么它是参考型？**
- 没有执行步骤（不是先 A 后 B）
- 没有输出模板（不要求固定格式）
- 不设 `disable-model-invocation`（Claude 可自动判断）
- 只读工具（在项目中写 API 时自动加载此规范）

---

## 核心框架

### Skills 与 SubAgents / CLAUDE.md 的选型

```
CLAUDE.md  → 每次都该知道的少量规则（<100行）→ 始终加载
Skills     → 特定场景下的详细指令和知识      → 按需加载
SubAgents  → 需要独立上下文的任务            → 隔离执行
```

> 💡 犹豫放 CLAUDE.md 还是 Skill → 放 Skill，并在 CLAUDE.md 里加一行引用。

### description 公式

```
[做什么] + [怎么做] + [什么时候用]
```

### 参考型 vs 任务型决策

```
这个 Skill 是规范/标准吗？
├── 是 → 参考型，不设 disable-model-invocation
└── 否 → 是一次明确的行动吗？
    ├── 是，且操作有风险 → 任务型，设 disable-model-invocation: true
    └── 是，但 Claude 可自行判断 → 任务型，不设禁用
```

---

## 评论区高价值讨论

### 🔥 1. Skills 为什么火了？

**读者 Jxin（4 赞）**：Skills 最大特色也是最大弊端——让模型自己决定何时使用什么技能。目前只有 CC 全家桶最稳定，其他模型效果不稳定。但 Skills 依然火了，火在**可分享、好理解、好生成**。

**作者答**：OpenClaw 生态推波助澜——Gateway 中介模式、声明式路由、SKILL.md 兼容，打通了分享和自动化的链条。

> 💡 Skills 不是技术突破，是**标准化**。把一个本来就存在的需求（分享可复用的 Agent 能力）用统一的文件格式规范下来，生态就开始裂变。

---

### 🔥 2. Java 后端开发适合做哪些 Skill？

**读者 Geek_f9766e（8 赞）**：后端 Java 开发整个流程应该分哪些 Skill？

**作者答**：不要先列 Skills 清单，先列**「你最痛的工作流」清单**。哪些事是每次都要重复教 Claude 的？这些痛点就是 Skills 的种子。大部分知识要么是公共知识（Claude 本来就会），要么是项目硬规范（放 CLAUDE.md），要么是文件类型规范（放 Rules）。Skills 是给**「任务驱动 + 知识密集」**的工作流准备的。

---

### 🔥 3. Sub-Agent 和 Skill 怎么区分？

**读者 ghostwritten（0 赞）**：看完 SubAgent 再看 Skills，脑子有点乱。

**作者答**：Sub-Agent 是**你的员工**，Skill 是**你交付给他的 SOP 手册**。员工有独立上下文、有工具有权限；手册告诉他遇到什么情况该怎么做。

> 💡 这个类比非常清晰——还记得我们在 SubAgent 专题里学的 `skills` 字段吗？就是「给子代理在启动时预加载 SOP 手册」。

---

### 🔥 4. 其他碎片

- **skill-creator**（爱海贼，6 赞）：Claude 内置了 skill-creator 技能，可以直接让它帮你创建 Skill——「凡是重复性的工作，都可以用 Skills 替代」
- **description 中英文**（Geek_62eb7b，3 赞）：全英文更好（GitHub 优质代码注释都是英文），但中文也可以
- **Skill 的存放与管理**（Sam Fu，0 赞）：项目级 vs 全局级的权衡——项目级优先，跨项目复用时考虑 Plugin 或移动到全局

---

## 相关链接

- 📁 [原文原始数据](../article-origin/011/)
- 📖 [Anthropic Agent Skills 公用仓库](https://github.com/anthropics/skills)
- 📝 [阅读指南](../000-阅读指南/)
