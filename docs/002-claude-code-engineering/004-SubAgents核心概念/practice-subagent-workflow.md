# 实践：用 Sub-Agent 搭建学习笔记自动化流水线

> 配套笔记：[004 · Sub-Agents 核心概念与应用价值](./README.md)
>
> 目标：把「拉文章→筛评论」两个高噪声环节做成 Sub-Agent，验证学到的隔离/约束/复用概念。

---

## 背景

我们学习每篇极客时间文章的标准流程：

```
① 拉文章 API → ② 存原文 → ③ 读 HTML 正文
→ ④ 写费曼笔记 → ⑤ 拉评论 API → ⑥ 筛评论
→ ⑦ 补充笔记 → ⑧ 更新学习清单 → ⑨ 提交推送
```

按 Sub-Agent 四场景判断：

| 环节 | 适合 Sub-Agent？ | 原因 |
|------|:---:|------|
| ① 拉文章 API | ✅ | 高噪声 curl 操作，中间输出对后续没价值 |
| ⑤ 拉评论 API | ✅ | 同上 |
| ⑥ 筛评论 | ✅ | 43 条→6 条，噪声比极高，筛选规则明确 |
| ④ 写笔记 | ❌ | 核心决策工作，需要理解全文脉络 |

结论：**①⑤⑥ 适合做成 Sub-Agent，④ 留在主对话。**

---

## 创建的两个 Sub-Agent

### Agent 1：article-fetcher（文章拉取专员）

**文件**：`.claude/agents/article-fetcher.md`

**设计要点**：

| 字段 | 配置 | 为什么 |
|------|------|--------|
| `tools` | Bash, Read, Write | 需要 curl 拉取 + 保存文件 |
| `model` | deepseek-chat | 拉数据是机械操作，用便宜模型省钱 |
| `description` | 包含「when user provides article URL and curl command」 | 明确触发场景 |

**职责**：
1. 接收文章 ID + curl 命令
2. 调用文章 API + 评论 API
3. 保存到 `article-origin/{序号}/`
4. 汇报结果摘要

### Agent 2：comment-filter（评论筛选员）

**文件**：`.claude/agents/comment-filter.md`

| 字段 | 配置 | 为什么 |
|------|------|--------|
| `tools` | Read, Bash | 只读评论文件 + 执行 Python 脚本 |
| `model` | deepseek-chat | 筛选规则明确，用便宜模型 |
| `description` | 包含「high-value」「informative discussions」 | 语义触发 |

**筛选规则**（设计为系统 prompt）：
- 点赞 >= 2 且内容有实质信息 → 保留
- 作者回复且信息量高 → 保留
- 纯打卡/赞美/催更 → 丢弃

---

## 使用方式

### 场景：学习新一篇文章

**Step 1**：用户提供文章链接和 curl 命令

**Step 2**：主对话派出 article-fetcher：

```
用 article-fetcher agent 拉取这篇文章：
- 文章: https://time.geekbang.org/column/article/XXXXXX
- curl: [浏览器复制的 curl 命令]
- 目标目录: docs/002-claude-code-engineering/article-origin/005/
```

Sub-Agent 独立执行 → 返回结果摘要 → 上下文释放。

**Step 3**：主对话读 HTML 正文，写费曼笔记

**Step 4**：主对话派出 comment-filter：

```
用 comment-filter agent 筛选评论：
- 文件: docs/002-claude-code-engineering/article-origin/005/comments.json
```

Sub-Agent 返回 6-10 条精华 → 主对话补充到笔记。

**Step 5**：提交推送

---

## 设计心得：为什么这样划分

### 1. 隔离在发挥作用

拉文章产生的 500 行 JSON、43 条评论的逐条解析——这些在 Sub-Agent 的独立上下文中完成，主对话只收到：

```
✅ 文章已拉取: 标题XXX, 正文16,000字
✅ 评论已筛选: 43条→6条精华
```

主对话上下文保持干净，用于写笔记这个核心决策任务。

### 2. 约束在发挥作用

comment-filter 的工具权限是 `Read, Bash`——它只能读文件和跑脚本，不能修改任何笔记内容。这就避免了「筛选员顺手写了笔记」的情况。笔记只能由主对话来写。

### 3. 复用在发挥作用

article-fetcher 的配置可以用于任何一篇极客时间文章，不需要每次重新教它怎么做。换一篇文章，只需要换文章 ID 和 curl 命令。

### 4. model 字段的价值

两个 Sub-Agent 都用 `deepseek-chat`（V3）而不是主对话的 `deepseek-v4-pro`——拉数据、筛评论不需要强推理能力，用便宜模型即可。正文里说的「增加 30-200% token，但可以靠选便宜模型反而省钱」，在实践里落到了实处。

---

## 与理论的对照

| 正文中学到的 | 实践中的对应 |
|-------------|------------|
| 隔离 = 上下文执行完即丢弃 | article-fetcher 的 curl 噪声不进入主对话 |
| 约束 = 工具权限边界 | comment-filter 只能 Read，不能 Write 笔记 |
| 复用 = 配置文件版本化 | agent 文件进 `.claude/agents/`，可跨项目复用 |
| 不嵌套 = 主对话编排 | 拉→筛→写→提交，主对话依次调度 |
| description = 最重要的字段 | 写了「when + what」，CC 能自动判断调用时机 |

---

## 局限与改进方向

**当前局限**：
- article-fetcher 依赖用户每次提供 curl（cookie 时效性问题）
- comment-filter 的筛选规则还比较机械，对「信息量」的判断不如人工精准

**后续可以做的**：
- 将筛选规则沉淀为 Skill，让筛选更智能
- 用 Hook 在 article-fetcher 写入文件前校验路径合法性
- 通过 Agent SDK 把整个流程编排成脚本
