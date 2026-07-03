# 012 · 10｜令行禁止：任务型 Skills、斜杠命令 /Command 实战

> 📖 原文出处：[极客时间 - 黄佳《Claude Code 工程化实战》](https://time.geekbang.org/column/article/946297)
>
> 📅 学习时间：2026-07-03

---

## 这篇文章在回答什么问题？

1. **任务型 Skill 和参考型 Skill 的核心区别是什么？** 一句话：`disable-model-invocation: true`。参考型是 Claude 自动判断何时加载，任务型是用户手动触发，禁止 Claude 自作主张。
2. **怎么给 Skill 传参？** 通过 `$ARGUMENTS`、`$1`/`$2` 等位置参数，让 `/commit fix login bug` 中的 `fix login bug` 动态注入到 Skill 内容中。
3. **怎么让 Skill 启动时就带着完整上下文？** 通过 `! \`command\`` 预处理机制——在 Skill 内容发给模型之前，先执行 shell 命令，把输出内联替换到 Prompt 中。
4. **怎么给任务型 Skill 加安全网？** Skill 级别的 Hooks——只在 Skill 生命周期内生效，调用完成自动清理。
5. **实战：三个团队标准命令怎么构建？** `/commit`（智能提交）、`/review`（代码审查）、`/pr-create`（创建 PR）。

---

## 原文转述

### 一、任务型 Skill = disable-model-invocation: true

上篇讲的是参考型 Skill——Claude 自动判断何时加载（如 API 设计规范）。这一篇讲另一面：**任务型 Skill**，核心就一行：

```yaml
disable-model-invocation: true
```

这行配置的意思是：**没有用户触发，Claude 绝不主动执行**。对于 commit、deploy、review 这类有副作用的操作，不允许 AI 自行决定是否执行。

**Skills vs Commands 历史**：早期 Commands 和 Skills 是两个独立组件。新版 CC 中 Commands 已合并到 Skills 成为子集。`commands/review.md` 和 `skills/review/SKILL.md` 都会创建 `/review`，但 Skill 优先。新建推荐用 Skills 目录（支持辅助文件）。

---

### 二、$ARGUMENTS 传参机制

```yaml
---
name: fix-issue
disable-model-invocation: true
argument-hint: [issue-number]
---

Fix GitHub issue $ARGUMENTS following our coding standards.
```

当用户输入 `/fix-issue 123`，`$ARGUMENTS` 被替换为 `123`。Claude 实际收到的内容是 `Fix GitHub issue 123 following our coding standards...`。

**两种参数方式**：

| 方式 | 变量 | 示例 |
|------|------|------|
| 单参数 | `$ARGUMENTS` | `/commit fix login bug` |
| 多参数 | `$1`, `$2` | `/pr-create "Add auth" "JWT"` |
| 位置 | `$ARGUMENTS[0]` 或 `$0` | `/migrate SearchBar React Vue` |

> 💡 如果 Skill 没定义 `$ARGUMENTS` 但用户传了参数，CC 会自动在末尾追加 `ARGUMENTS: <用户输入>`——不丢参。

---

### 三、! `command` 动态上下文注入（核心技术）

这是任务型 Skill 最巧妙的设计。当用户输入 `/pr-create "Add auth"`，模型不知道当前分支、commit 记录、文件变更。传统做法需要**先花多轮工具调用去收集这些信息**。

**! `command` 是预处理器**——在 Skill 内容发送给模型之前，先在 shell 执行命令，把输出内联替换到 Prompt 中：

```markdown
Current branch:
!`git branch --show-current`

Recent commits:
!`git log origin/main..HEAD --oneline`

Files changed:
!`git diff --stat origin/main`
```

Claude **实际收到的 Prompt**（替换后）：

```markdown
Current branch:
feature/auth

Recent commits:
a1b2c3d Add JWT middleware
d4e5f6g Add login endpoint

Files changed:
 src/auth/middleware.ts | 45 +++
 src/auth/login.ts     | 82 +++
 3 files changed, 161 insertions(+)
```

> 💡 效果：Claude 启动时就拥有完整上下文，**直接生成 PR 标题和描述，无需额外工具调用**。减少 3-5 次工具调用，提升响应一致性。

⚠️ **安全注意**：`$ARGUMENTS` 会进入 shell 命令，必须在 `allowed-tools` 中严格限制可执行范围。

---

### 四、Skill 级别的 Hooks

任务型 Skill 执行的是有副作用的操作（提交代码、部署），需要安全网。Hooks 通过三层树形结构定义：`事件 → 匹配规则 → 命令列表`：

```yaml
hooks:
  PreToolUse:                    # 事件：工具使用前
    - matcher: Bash              # 匹配：Bash 工具
      hooks:
        - type: command
          command: echo "$TOOL_INPUT" >> deploy.log  # 动作：记录日志
  PostToolUse:                   # 事件：工具使用后
    - matcher: Edit
      hooks:
        - type: command
          command: npx prettier --write "$FILE_PATH"  # 动作：自动格式化
```

**Skill Hooks vs 全局 Hooks**：Skill Hooks 仅在 Skill 激活时生效，调用完成自动清理；全局 Hooks 对所有对话生效。

---

### 五、七步设计清单

```
1. 动作是什么？    → 命名（commit / deploy / review）
2. 谁能触发？      → disable-model-invocation: true
3. 需要什么权限？  → allowed-tools 精确到命令级
4. 启动时上下文？  → !`command` 预注入
5. 安全网？        → hooks
6. 输出量大不大？  → 大则 context: fork（子代理隔离）
7. 用什么模型？    → 简单 haiku / 复杂 sonnet
```

**核心原则**：
- **单一职责**：`/commit`、`/push`、`/review` 分开，不搞 `/git-all-in-one`
- **权限最小化**：`Bash(git status:*, git add:*, git commit:*)` 不写 `Bash(*)`
- **参数有意义**：`argument-hint: [commit message]` 不写 `[args]`

---

### 六、三个实战命令

**① `/commit`**——智能提交

```yaml
disable-model-invocation: true
allowed-tools: Bash(git status:*), Bash(git add:*), Bash(git commit:*), Bash(git diff:*)
model: haiku
```

有参数用参数，没参数自动从 diff 生成 conventional commits 格式消息。haiku 够用且快。

**② `/review`**——代码审查

```yaml
disable-model-invocation: true
allowed-tools: Read, Grep, Glob, Bash(git diff:*)
```

只读权限（无 Edit/Write），输出结构化报告：Critical > Warning > Suggestion 三级。

**③ `/pr-create`**——创建 PR

```yaml
disable-model-invocation: true
allowed-tools: Bash(git:*), Bash(gh:*)
```

使用 `! \`command\`` 预注入分支名、commit 记录、文件变更，Claude 启动即有完整上下文。

**命名空间组织**：`/git:status`、`/git:log`、`/test:unit`、`/test:e2e`——目录结构即命名空间。

---

### 七、任务型 vs 参考型 vs SubAgent 三方对比

黄佳老师以「代码审查」为例做了三方对比：

| 维度 | 任务型 Skill (/review) | 参考型 Skill | SubAgent |
|------|---------------------|------------|---------|
| **触发** | 用户手动 `/review` | Claude 自动判断 | 主对话委派或自动 |
| **上下文** | 共享主对话 | 共享主对话 | 独立上下文 |
| **权限** | allowed-tools 约束 | allowed-tools 约束 | tools 白名单 |
| **适合** | 标准化操作、需手动触发 | 知识自动到场 | 需隔离、高噪声 |

> 💡 三者可以共存。一个成熟的团队工具箱：参考型负责知识自动到场，任务型负责手动快捷操作，SubAgent 负责隔离和高噪声。

---

## 核心框架

### 任务型 Skill 速查公式

```
任务型 Skill = disable-model-invocation: true
              + allowed-tools（精确到命令级）
              + !`command`（预注入上下文）
              + hooks（Skill 级安全网）
              + $ARGUMENTS（参数传递）
```

### 七步设计清单

```
动作 → 触发 → 权限 → 上下文 → 安全网 → 隔离 → 模型
```

### 三种能力扩展统一视角

```
参考型 Skill：   Claude 自动判断 → 知识在需要时到场
任务型 Skill：   用户手动触发 → 重复流程变快捷方式
SubAgent：      隔离上下文 → 高噪声/独立权限任务
```

---

## 相关链接

- 📁 [原文原始数据](../article-origin/012/)
- 📝 [上一讲：SKILL.md 结构与触发机制](../011-SKILL-md结构与触发机制/)

---

## 评论区高价值讨论

### 🔥 1. Skills 目录不支持命名空间

**读者 :-p（5 赞）**：Commands 可以通过目录划分命名空间（`git:commit`），Skills 目录能吗？比如 `.claude/skills/git/commit/SKILL.md`？

**作者答**：不行。Skills 的 `<skill-name>` 就是目录名本身，不支持多层级命名空间。写成 `.claude/skills/git/commit/SKILL.md`，CC 只会把 `commit` 当成 skill name，`git/` 只是普通父目录。

> 💡 这是 Skills 和 Commands 之间的一个重要差异——Commands 支持目录命名空间，Skills 暂时不支持。

---

### 🔥 2. 实战分享：两个实用 Skill

**读者 和尚（4 赞）**：分享两个工作中常用的 Skill：
1. **save-webpage**：把网页抓取保存为 Markdown，支持登录页面和懒加载图片。触发场景：用户说「保存网页」「网页转 markdown」
2. **mermaid-to-png**：将 Markdown 中的 Mermaid 图表转为高分辨率 PNG。触发场景：「mermaid 转 png」「架构图导出」

---

### 🔥 3. 参考型 Skill 和 wiki 文档怎么选？

**读者 smiler（2 赞）**：想让 Claude 理解复杂的业务上下文（如 PRD 中的下载功能涉及多个 API 交互），应该写参考型 Skill 还是 wiki 文档？

**作者答**：直接把文档放在项目参考目录里就行，CC 需要时会自己查找。不要手动喂——让他自己找，或者用参考型 Skill 渐进加载。

---

### 🔥 4. 实用碎片

- **`/! command` 中的 GML**（kkxue）：笔误，应该是 GLM（智谱模型）
- **`disable-model-invocation` 的意义**（jssfy）：不是「更灵活」，是为了**防止随意触发**——把 Skill 变成需要明确授权的命令
- **Session ID 日志**（迪沃斯托克）：`~/.claude/projects/{project-hash}/{session-id}.jsonl`，子代理日志在同名目录的 `subagents/` 下，可用于审计追溯
