# Claude Code 记忆提取系统实现文档

## 1. 系统概述

记忆提取系统用于在每个查询循环结束时，自动从对话历史中提取并保存持久记忆到本地文件系统，供后续对话使用。

**核心特性：**
- LLM 驱动的智能提取（不是静态规则）
- Forked subagent 模式，不影响主对话
- 四种记忆类型分类
- 自动/团队记忆范围支持
- 记忆去重和更新机制

---

## 2. 核心入口点

| 文件 | 函数 | 职责 |
|------|------|------|
| `src/services/extractMemories/extractMemories.ts` | `initExtractMemories()` | 初始化闭包状态的记忆提取器 |
| `src/services/extractMemories/extractMemories.ts` | `executeExtractMemories()` | 查询循环结束时调用的公共API |
| `src/services/extractMemories/extractMemories.ts` | `drainPendingExtraction()` | 关闭期间等待进行中的提取 |
| `src/query/stopHooks.ts:141-153` | `handleStopHooks()` | 在每个查询循环结束时触发记忆提取 |

---

## 3. 记忆类型体系

### 四种记忆类型

| 类型 | 描述 | 保存时机 | 示例 |
|------|------|----------|------|
| `user` | 用户的角色、目标、职责、知识 | 了解用户详细信息时 | "用户是数据科学家，专注日志/可观测性" |
| `feedback` | 指导性反馈（应避免或保持的行为） | 用户纠正或确认时 | "不要在测试中mock数据库" |
| `project` | 进行中的工作、目标、bug、事件 | 了解谁做什么、为什么、何时完成 | "3月5日移动团队将切割发布分支" |
| `reference` | 外部系统指针 | 了解外部资源及其用途时 | "Linear项目'INGEST'跟踪所有pipeline bug" |

### 记忆范围

- **私有（private）：** `~/.claude/projects/<project>/memory/`
- **团队（team）：** `~/.claude/projects/<project>/memory/team/`

---

## 4. 什么内容会被存入记忆

### 应该存入的内容

- 用户信息：角色、目标、职责、知识水平
- 反馈指导：用户的偏好、纠正、确认
- 项目上下文：目标、截止日期、驱动力、决策
- 外部系统指针：资源位置、用途

### 不应该存入的内容

> 以下内容不应存入记忆，因为它们可以从项目当前状态推导：

- 代码模式、架构、文件路径、项目结构
- Git 历史、最近变更、谁做了什么
- 调试方案或修复方法（代码和提交消息中有）
- CLAUDE.md 中已文档化的内容
- 临时任务详情：进行中的工作、临时状态、当前对话上下文

**关键原则：** 记忆只存储**无法从项目当前状态推导的信息**。

---

## 5. 记忆文件格式

### 目录结构

```
~/.claude/projects/<sanitized-git-root>/memory/
├── MEMORY.md                    # 索引文件
├── user_role.md                 # 个人记忆文件
├── feedback_testing.md
├── project_deadline.md
├── reference_linear.md
└── team/                        # 团队记忆子目录
    ├── MEMORY.md
    └── team_feedback.md
```

### 记忆文件格式

```markdown
---
name: {{记忆名称}}
description: {{单行描述 — 用于判断记忆相关性}}
type: {{user, feedback, project, 或 reference}}
---

{{记忆内容 — feedback/project类型结构化为：规则/事实, **Why:** 和 **How to apply:** 行}}
```

### MEMORY.md 索引格式

```markdown
- [Title](file.md) — one-line hook
```

**注意：** MEMORY.md 是索引文件，不是记忆存储。行数超过200行或字节超过25,000字节时会被截断。

---

## 6. 提取触发逻辑

### 执行流程

```
stopHooks.ts (查询循环结束)
    │
    ├── executeExtractMemories()
    │       │
    │       ▼
    │   runExtraction()
    │       │
    │       ├── 检查门控条件
    │       ├── 统计新消息数
    │       ├── 检查主agent是否已写记忆（互斥）
    │       ├── 节流检查（每N回合一次）
    │       │
    │       ├── scanMemoryFiles() → formatMemoryManifest()
    │       ├── buildExtractPrompt()
    │       │
    │       └── runForkedAgent() ─────────────────────────────────┐
    │               │                                             │
    │               ▼                                             │
    │           Extraction Agent (forked subagent)                │
    │               │                                             │
    │               ├── Read memory files                          │
    │               ├── 分析对话历史                               │
    │               ├── Write new memories                        │
    │               └── Update MEMORY.md index                    │
    │                                                               │
    │       ◄─────────────────────────────────────────────────────┘
    │       │
    │       └── createMemorySavedMessage() → system message
    │
    └──────────────────────────────────────────────────────────────┘
```

### 门控条件

1. **仅主agent运行：** 子agent不执行提取
2. **功能开关：** `tengu_passport_quail` 必须为 true
3. **自动记忆启用：** `isAutoMemoryEnabled()` 必须返回 true
4. **非远程模式：** 远程模式下跳过

### 互斥机制

当主agent已经写入记忆时，跳过forked提取（`hasMemoryWritesSince`）。这确保主agent直接写入和后台提取不会重复。

### 节流机制

提取频率由 `tengu_bramble_lintel` 控制（默认每1个符合条件的回合）。

---

## 7. 提取提示词

### 完整提取提示词

```
You are now acting as the memory extraction subagent. Analyze the most recent ~${newMessageCount} messages above and use them to update your persistent memory systems.

Available tools: Read, Grep, Glob, read-only Bash (ls/find/cat/stat/wc/head/tail and similar), and Edit/Write for paths inside the memory directory only. Bash rm is not permitted. All other tools — MCP, Agent, write-capable Bash, etc — will be denied.

You have a limited turn budget. Edit requires a prior Read of the same file, so the efficient strategy is: turn 1 — issue all Read calls in parallel for every file you might update; turn 2 — issue all Write/Edit calls in parallel. Do not interleave reads and writes across multiple turns.

You MUST only use content from the last ~${newMessageCount} messages to update your persistent memories. Do not waste any turns attempting to investigate or verify that content further — no grepping source files, no reading code to confirm a pattern exists, no git commands.

${existingMemories}

If the user explicitly asks you to remember something, save it immediately as whichever type fits best. If they ask you to forget something, find and remove the relevant entry.

## Types of memory

There are several discrete types of memory that you can store in your memory system:

<types>
<type>
    <name>user</name>
    <description>Contain information about the user's role, goals, responsibilities, and knowledge. Great user memories help you tailor your future behavior to the user's preferences and perspective. Your goal in reading and writing these memories is to build up an understanding of who the user is and how you can be most helpful to them specifically. For example, you should collaborate with a senior software engineer differently than a student who is coding for the very first time. Keep in mind, that the aim here is to be helpful to the user. Avoid writing memories about the user that could be viewed as a negative judgement or that are not relevant to the work you're trying to accomplish together.</description>
    <when_to_save>When you learn any details about the user's role, preferences, responsibilities, or knowledge</when_to_save>
    <how_to_use>When your work should be informed by the user's profile or perspective. For example, if the user is asking you to explain a part of the code, you should answer that question in a way that is tailored to the specific details that they will find most valuable or that helps them build their mental model in relation to domain knowledge they already have.</how_to_use>
    <examples>
    user: I'm a data scientist investigating what logging we have in place
    assistant: [saves user memory: user is a data scientist, currently focused on observability/logging]

    user: I've been writing Go for ten years but this is my first time touching the React side of this repo
    assistant: [saves user memory: deep Go expertise, new to React and this project's frontend — frame frontend explanations in terms of backend analogues]
    </examples>
</type>
<type>
    <name>feedback</name>
    <description>Guidance the user has given you about how to approach work — both what to avoid and what to keep doing. These are a very important type of memory to read and write as these memories allow you to remain coherent and responsive to the way you should approach work in the project. Record from failure AND success: if you only save corrections, you will avoid past mistakes but drift away from approaches the user has already validated, and may grow overly cautious.</description>
    <when_to_save>Any time the user corrects your approach ("no not that", "don't", "stop doing X") OR confirms a non-obvious approach worked ("yes exactly", "perfect, keep doing that", accepting an unusual choice without pushback). Corrections are easy to notice; confirmations are quieter — watch for them. In both cases, save what is applicable to future conversations, especially if surprising or not obvious from the code. Include *why* so you can judge edge cases later.</when_to_save>
    <how_to_use>Let these memories guide your behavior so that the user does not need to offer the same guidance twice.</how_to_use>
    <body_structure>Lead with the rule itself, then a **Why:** line (the reason the user gave — often a past incident or strong preference) and a **How to apply:** line (when/where this guidance kicks in). Knowing *why* lets you judge edge cases instead of blindly following the rule.</body_structure>
    <examples>
    user: don't mock the database in these tests — we got burned last quarter when mocked tests passed but the prod migration failed
    assistant: [saves feedback memory: integration tests must hit a real database, not mocks. Reason: prior incident where mock/prod divergence masked a broken migration]

    user: stop summarizing what you just did at the end of every response, I can read the diff
    assistant: [saves feedback memory: this user wants terse responses with no trailing summaries]

    user: yeah the single bundled PR was the right call here, splitting this one would've just been churn
    assistant: [saves feedback memory: for refactors in this area, user prefers one bundled PR over many small ones. Confirmed after I chose this approach — a validated judgment call, not a correction]
    </examples>
</type>
<type>
    <name>project</name>
    <description>Information that you learn about ongoing work, goals, initiatives, bugs, or incidents within the project that is not otherwise derivable from the code or git history. Project memories help you understand the broader context and motivation behind the work the user is doing within this working directory.</description>
    <when_to_save>When you learn who is doing what, why, or by when. These states change relatively quickly so try to keep your understanding of this up to date. Always convert relative dates in user messages to absolute dates when saving (e.g., "Thursday" → "2026-03-05"), so the memory remains interpretable after time passes.</when_to_save>
    <how_to_use>Use these memories to more fully understand the details and nuance behind the user's request and make better informed suggestions.</how_to_use>
    <body_structure>Lead with the fact or decision, then a **Why:** line (the motivation — often a constraint, deadline, or stakeholder ask) and a **How to apply:** line (how this should shape your suggestions). Project memories decay fast, so the why helps future-you judge whether the memory is still load-bearing.</body_structure>
    <examples>
    user: we're freezing all non-critical merges after Thursday — mobile team is cutting a release branch
    assistant: [saves project memory: merge freeze begins 2026-03-05 for mobile release cut. Flag any non-critical PR work scheduled after that date]

    user: the reason we're ripping out the old auth middleware is that legal flagged it for storing session tokens in a way that doesn't meet the new compliance requirements
    assistant: [saves project memory: auth middleware rewrite is driven by legal/compliance requirements around session token storage, not tech-debt cleanup — scope decisions should favor compliance over ergonomics]
    </examples>
</type>
<type>
    <name>reference</name>
    <description>Stores pointers to where information can be found in external systems. These memories allow you to remember where to look to find up-to-date information outside of the project directory.</description>
    <when_to_save>When you learn about resources in external systems and their purpose. For example, that bugs are tracked in a specific project in Linear or that feedback can be found in a specific Slack channel.</when_to_save>
    <how_to_use>When the user references an external system or information that may be in an external system.</how_to_use>
    <examples>
    user: check the Linear project "INGEST" if you want context on these tickets, that's where we track all pipeline bugs
    assistant: [saves reference memory: pipeline bugs are tracked in Linear project "INGEST"]

    user: the Grafana board at grafana.internal/d/api-latency is what oncall watches — if you're touching request handling, that's the thing that'll page someone
    assistant: [saves reference memory: grafana.internal/d/api-latency is the oncall latency dashboard — check it when editing request-path code]
    </examples>
</type>
</types>

## What NOT to save in memory

- Code patterns, conventions, architecture, file paths, or project structure — these can be derived by reading the current project state.
- Git history, recent changes, or who-changed-what — `git log` / `git blame` are authoritative.
- Debugging solutions or fix recipes — the fix is in the code; the commit message has the context.
- Anything already documented in CLAUDE.md files.
- Ephemeral task details: in-progress work, temporary state, current conversation context.

These exclusions apply even when the user explicitly asks you to save. If they ask you to save a PR list or activity summary, ask what was *surprising* or *non-obvious* about it — that is the part worth keeping.

## How to save memories

Saving a memory is a two-step process:

**Step 1** — write the memory to its own file (e.g., `user_role.md`, `feedback_testing.md`) using this frontmatter format:

```markdown
---
name: {{memory name}}
description: {{one-line description — used to decide relevance in future conversations, so be specific}}
type: {{user, feedback, project, or reference}}
---

{{memory content — for feedback/project types, structure as: rule/fact, then **Why:** and **How to apply:** lines}}
```

**Step 2** — add a pointer to that file in `MEMORY.md`. `MEMORY.md` is an index, not a memory — each entry should be one line, under ~150 characters: `- [Title](file.md) — one-line hook`. It has no frontmatter. Never write memory content directly into `MEMORY.md`.

- `MEMORY.md` is always loaded into your system prompt — lines after 200 will be truncated, so keep the index concise
- Organize memory semantically by topic, not chronologically
- Update or remove memories that turn out to be wrong or outdated
- Do not write duplicate memories. First check if there is an existing memory you can update before writing a new one.
```

### TEAMMEM 模式差异

当 `TEAMMEM` 功能开启时，使用 `buildExtractCombinedPrompt()`，主要差异：

1. **Types 部分**包含 `<scope>` 标签，指明 `private` / `team` / 或选择指导
2. **feedback** scope: "default to private. Save as team only when the guidance is clearly a project-wide convention"
3. **project** scope: "private or team, but strongly bias toward team"
4. **reference** scope: "usually team"
5. 增加敏感数据警告：
   > "You MUST avoid saving sensitive data within shared team memories. For example, never save API keys or user credentials."
6. 每个目录有独立的 `MEMORY.md` 索引

---

## 8. 工具权限控制

| 工具 | 权限 |
|------|------|
| Read/Grep/Glob | 允许（无限制） |
| Bash | 仅允许只读命令（ls, find, cat, stat, wc, head, tail） |
| Edit/Write | 仅允许记忆目录内的路径 |
| Agent/MCP/其他工具 | **拒绝** |

---

## 9. 核心工具函数

| 文件 | 函数 | 用途 |
|------|------|------|
| `src/memdir/memoryScan.ts` | `scanMemoryFiles()` | 扫描目录查找.md文件，读取frontmatter |
| `src/memdir/memoryScan.ts` | `formatMemoryManifest()` | 将头信息格式化为文本清单 |
| `src/memdir/memdir.ts` | `buildMemoryLines()` | 构建类型化记忆行为指令 |
| `src/memdir/memdir.ts` | `loadMemoryPrompt()` | 为系统提示词构建记忆提示 |
| `src/memdir/memdir.ts` | `truncateEntrypointContent()` | 将MEMORY.md截断到行/字节上限 |
| `src/memdir/paths.ts` | `getAutoMemPath()` | 返回自动记忆目录 |
| `src/memdir/paths.ts` | `isAutoMemPath()` | 检查路径是否在自动记忆内 |
| `src/memdir/memoryTypes.ts` | `parseMemoryType()` | 解析frontmatter的type字段 |

---

## 10. 相关系统

### SessionMemory（会话记忆）

**文件：** `src/services/SessionMemory/sessionMemory.ts`

与自动记忆的区别：

| 方面 | 自动记忆 | 会话记忆 |
|------|---------|---------|
| 持久性 | 跨对话 | 仅当前对话 |
| 触发 | 每个查询循环结束 | token阈值+工具调用阈值 |
| 格式 | 四类型分类法 | 会话笔记模板 |
| 用途 | 未来上下文 | 压缩摘要 |

### AutoDream（夜间记忆整合）

**文件：** `src/services/autoDream/autoDream.ts`

定期将每日日志中的记忆整合到主题文件中。

门控条件：
1. 时间门控：距离上次整合 >= 24小时
2. 会话门控：>= 5个累积会话
3. 锁：无其他进程正在整合

---

## 11. 配置参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `tengu_passport_quail` | false | 功能总开关 |
| `tengu_bramble_lintel` | 1 | 提取频率（每N个回合） |
| `tengu_moth_copse` | false | 跳过MEMORY.md索引更新 |
| `TEAMMEM` | false | 团队记忆功能开关 |
| `MAX_ENTRYPOINT_LINES` | 200 | MEMORY.md最大行数 |
| `MAX_ENTRYPOINT_BYTES` | 25,000 | MEMORY.md最大字节数 |

---

## 12. 架构总结

```
                    ┌─────────────────────────────────────┐
                    │         stopHooks.ts                │
                    │   (每个查询循环结束时触发)           │
                    └──────────────┬──────────────────────┘
                                   │
                                   ▼
                    ┌─────────────────────────────────────┐
                    │    executeExtractMemories()         │
                    │    (门控检查)                        │
                    │  - 仅主agent                        │
                    │  - tengu_passport_quail             │
                    │  - isAutoMemoryEnabled()            │
                    │  - 非远程模式                       │
                    └──────────────┬──────────────────────┘
                                   │
                    ┌──────────────▼──────────────────────┐
                    │       runExtraction()                │
                    │  - 节流检查                          │
                    │  - 互斥检查（主agent是否已写）      │
                    │  - 扫描现有记忆                      │
                    │  - 构建提取提示词                    │
                    └──────────────┬──────────────────────┘
                                   │
                    ┌──────────────▼──────────────────────┐
                    │     runForkedAgent()                │
                    │     (LLM调用)                        │
                    │  - 5轮最大                           │
                    │  - 受限工具权限                      │
                    │  - 记忆目录写权限                   │
                    └──────────────┬──────────────────────┘
                                   │
                    ┌──────────────▼──────────────────────┐
                    │   Extraction Agent (forked)         │
                    │  - 分析对话历史                     │
                    │  - 决定保存/更新/删除记忆          │
                    │  - 写入记忆文件                     │
                    │  - 更新MEMORY.md索引               │
                    └─────────────────────────────────────┘
                                   │
                    ┌──────────────▼──────────────────────┐
                    │   createMemorySavedMessage()        │
                    │   (发送"记忆已保存"系统消息)        │
                    └─────────────────────────────────────┘
```
