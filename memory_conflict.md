# AI Agent 记忆冲突机制调研

> 调研日期：2026年4月22日
> 调研对象：Claude Code（源码分析）、Hermes Agent、OpenClaw
> 调研核心问题：当多次记忆之间产生矛盾时，各 Agent 的记忆系统如何处理？

---

## 一、Claude Code 记忆系统全貌

### 1.1 记忆的存储结构

Claude Code 使用**纯文件系统**作为记忆后端。每条记忆是一个独立的 `.md` 文件，配有 YAML frontmatter；所有记忆通过一个 `MEMORY.md` 索引文件串联。

```
~/.claude/projects/<project-slug>/memory/              ← private 记忆目录
    MEMORY.md                                           ← 索引文件（每行一个指针）
    user_role.md                                        ← 一条记忆
    feedback_testing.md                                 ← 一条记忆
    project_deadline.md                                 ← 一条记忆

~/.claude/projects/<project-slug>/memory/team/          ← team 记忆目录
    MEMORY.md                                           ← 团队索引文件
    feedback_code_style.md
    reference_linear.md
```

单条记忆文件的格式：

```markdown
---
name: testing policy
description: Integration tests must use real database — team convention
type: feedback
---

Integration tests must hit a real database, not mocks.
**Why:** Prior incident where mock/prod divergence masked a broken migration.
**How to apply:** Any test file under tests/integration/ should connect to the test DB.
```

`MEMORY.md` 索引文件只存单行指针，不存内容：

```markdown
- [Testing policy](feedback_testing.md) — integration tests use real DB, not mocks
- [User role](user_role.md) — senior backend engineer, new to React
```

**源码出处**：`src/memdir/memdir.ts` 中的 `buildMemoryLines()` 函数定义了完整的 prompt 指令，包括如何保存、索引的行数限制（`MAX_ENTRYPOINT_LINES = 200`）、字节上限（`MAX_ENTRYPOINT_BYTES = 25,000`）等。

### 1.2 Private 记忆 vs Team 记忆

| 维度 | Private 记忆 | Team 记忆 |
|------|-------------|-----------|
| **存储位置** | `~/.claude/projects/<slug>/memory/` | `~/.claude/projects/<slug>/memory/team/` |
| **可见范围** | 仅当前用户 | 同一项目目录下的所有用户共享 |
| **同步机制** | 无（本地读写） | 每个 session 开始时同步 |
| **典型内容** | 用户个人偏好、个人角色信息 | 项目规范、团队约定、外部资源引用 |
| **启用条件** | 默认启用（`isAutoMemoryEnabled()`） | 需要 feature flag `tengu_herring_clock` 开启 |

**源码出处**：
- Team 记忆路径定义：`src/memdir/teamMemPaths.ts` → `getTeamMemPath()` 返回 `join(getAutoMemPath(), 'team')`
- 启用检查：`isTeamMemoryEnabled()` 要求 auto memory 已启用 + feature flag 开启
- 两目录模式的 prompt 构建：`src/memdir/teamMemPrompts.ts` → `buildCombinedMemoryPrompt()`
- 单目录模式的 prompt 构建：`src/memdir/memdir.ts` → `buildMemoryLines()`
- 路由逻辑：`src/memdir/memdir.ts` → `loadMemoryPrompt()` 中根据 `isTeamMemoryEnabled()` 选择走哪条分支

当 team 记忆未启用时，系统走 `TYPES_SECTION_INDIVIDUAL` 分支，所有记忆只有一个目录，没有 scope 概念，也没有 private/team 矛盾检查。

### 1.3 四种记忆类型

源码定义（`src/memdir/memoryTypes.ts`）：

```typescript
export const MEMORY_TYPES = ['user', 'feedback', 'project', 'reference'] as const
```

#### (1) user — 用户画像

- **scope**：always private（永远私有）
- **存什么**：用户的角色、目标、职责、知识水平。目标是构建对用户的理解，以便针对性协作
- **何时存**：了解到用户的角色、偏好、职责或知识水平时
- **何时不存**：对用户的负面评判、与工作无关的个人信息
- **矛盾处理**：❌ 无显式矛盾检查指令
- **示例**：
  - 用户说"我是数据科学家" → 存 `user is a data scientist, currently focused on observability/logging`
  - 用户说"写了十年 Go，第一次碰 React" → 存 `deep Go expertise, new to React — frame frontend explanations in terms of backend analogues`

#### (2) feedback — 行为反馈

- **scope**：默认 private；仅当是项目级公约（如测试策略、构建规范）时存为 team
- **存什么**：用户对 Claude 工作方式的纠正**和确认**。强调"不只记录错误修正，也要记录确认有效的做法"
- **何时存**：
  - 用户纠正时（"别这么做"、"停止做X"）
  - 用户确认非显而易见的做法有效时（"对就是这样"、"完美"、无异议地接受非常规选择）
- **何时不存**：显而易见的事（能从代码里推导出来的）
- **必须包含**：`Why:` 行（原因）+ `How to apply:` 行（应用时机）——记录原因是为了在边界情况下能做判断，而不是盲目遵循规则
- **矛盾处理**：✅ **唯一有显式跨 scope 矛盾检查的类型**

  源码原文：
  > `"Before saving a private feedback memory, check that it doesn't contradict a team feedback memory — if it does, either don't save it or note the override explicitly."`

- **示例**：
  - 用户说"别 mock 数据库" → 存 team feedback：`integration tests must hit a real database, not mocks. Reason: prior incident where mock/prod divergence masked a broken migration`
  - 用户说"别在回复末尾总结" → 存 private feedback：`this user wants terse responses with no trailing summaries. Private because it's a communication preference, not a project convention`
  - 用户确认"对，一个大 PR 就行" → 存 private feedback：`for refactors in this area, user prefers one bundled PR over many small ones. Confirmed after I chose this approach — a validated judgment call, not a correction`

#### (3) project — 项目动态

- **scope**：private 或 team，强烈倾向 team
- **存什么**：项目进行中的工作、目标、截止日期、事件——代码/git 历史无法推导的部分
- **何时存**：了解到谁在做什么、为什么、截止何时。**必须将相对日期转为绝对日期**（"周四" → "2026-03-05"）
- **何时不存**：Git 能查到的变更历史、已在 CLAUDE.md 中记录的信息
- **必须包含**：`Why:` 行 + `How to apply:` 行。"Project memories decay fast, so the why helps future-you judge whether the memory is still load-bearing."
- **矛盾处理**：❌ 无显式矛盾检查，但要求 `"These states change relatively quickly so try to keep your understanding of this up to date"` —— 期望主动覆盖旧信息
- **示例**：
  - 用户说"周四后冻结非关键合并" → 存 team：`merge freeze begins 2026-03-05 for mobile release cut`
  - 用户说"正在拆掉旧 auth 中间件是因为合规问题" → 存 team：`auth middleware rewrite is driven by legal/compliance, not tech-debt cleanup`

#### (4) reference — 外部引用

- **scope**：通常 team
- **存什么**：外部系统中信息的指针（Linear 项目、Grafana 看板、Slack 频道等）
- **何时存**：了解到外部系统资源及其用途时
- **矛盾处理**：❌ 无显式矛盾检查
- **示例**：
  - 用户说"管道 bug 在 Linear 的 INGEST 项目里" → 存 team：`pipeline bugs are tracked in Linear project "INGEST"`
  - 用户说"这个 Grafana 看板是 oncall 盯的" → 存 team：`grafana.internal/d/api-latency is the oncall latency dashboard`

#### 为什么只有 feedback 有显式矛盾检查？

feedback 是唯一一种 **private 和 team 可能同时存在、且语义可能直接对冲的类型**。典型冲突：team 说"4 空格缩进"，个人 private 说"我喜欢 2 空格"。其他类型要么永远 private（user），要么几乎永远 team（reference），scope 间不太会矛盾。

但这只是**写入时**的检查。**读取时**的矛盾处理（Memory Drift Caveat）是全类型通用的。

### 1.4 完整的记忆更新策略（五层防御）

Claude Code 对记忆冲突的处理不是单一机制，而是**五层防御**：

#### 第一层：写入时去重（Write-time dedup）

**源码位置**：`src/memdir/memdir.ts` → `buildMemoryLines()` → How to save memories

```
- Do not write duplicate memories. First check if there is an existing memory
  you can update before writing a new one.
- Update or remove memories that turn out to be wrong or outdated
```

机制：`MEMORY.md` 索引在每个 session 开始时自动注入到 context。Claude 在写入新记忆前，会先扫描索引判断是否已有同主题记忆。如果有，就用 FileEditTool 编辑已有文件而非创建新文件。

#### 第二层：跨 Scope 矛盾检查（Cross-scope contradiction check）

**源码位置**：`src/memdir/memoryTypes.ts` → `TYPES_SECTION_COMBINED` → feedback 类型

仅对 feedback 类型生效：在存 private feedback 之前，检查 team feedback 是否有相悖的规则。如果有，要么不存，要么显式标注覆盖关系。

#### 第三层：陈旧标注（Freshness tagging）

**源码位置**：`src/memdir/memoryAge.ts`

```typescript
export function memoryFreshnessText(mtimeMs: number): string {
  const d = memoryAgeDays(mtimeMs)
  if (d <= 1) return ''
  return (
    `This memory is ${d} days old. ` +
    `Memories are point-in-time observations, not live state — ` +
    `claims about code behavior or file:line citations may be outdated. ` +
    `Verify against current code before asserting as fact.`
  )
}
```

超过 1 天的记忆被读取时，自动附加"这条记忆是 X 天前的"警告。模型在输出前需验证。

#### 第四层：读取时漂移检测（Read-time drift caveat）

**源码位置**：`src/memdir/memoryTypes.ts` → `MEMORY_DRIFT_CAVEAT`

```
Memory records can become stale over time. Use memory as context for what was
true at a given point in time. Before answering the user or building assumptions
based solely on information in memory records, verify that the memory is still
correct and up-to-date by reading the current state of the files or resources.
If a recalled memory conflicts with current information, trust what you observe
now — and update or remove the stale memory rather than acting on it.
```

核心策略：**当前观察 > 记忆内容**。如果记忆与当前状态矛盾，信任当前状态，同时清理旧记忆。

#### 第五层：使用前验证（Pre-recommendation verification）

**源码位置**：`src/memdir/memoryTypes.ts` → `TRUSTING_RECALL_SECTION`

```
A memory that names a specific function, file, or flag is a claim that it
existed *when the memory was written*. It may have been renamed, removed,
or never merged. Before recommending it:

- If the memory names a file path: check the file exists.
- If the memory names a function or flag: grep for it.
- If the user is about to act on your recommendation, verify first.

"The memory says X exists" is not the same as "X exists now."
```

### 1.5 具体场景演练：用户偏好变更

**场景**：1h 前用户说"我喜欢选项 A"，现在说"我喜欢选项 B"。

#### 1h 前——首次写入

Claude 判断为 **user 类型**记忆（用户偏好），执行标准两步写入：

**Step 1**：FileWriteTool 创建记忆文件

```
文件：~/.claude/projects/<project>/memory/preference_option.md

---
name: option preference
description: User prefers option A for feature X
type: user
---

User prefers option A for feature X.
```

**Step 2**：FileEditTool 在 `MEMORY.md` 追加索引行

```
- [Option preference](preference_option.md) — user prefers option A
```

#### 现在——偏好变更

用户说"其实我还是喜欢选项 B"。此时 `MEMORY.md` 索引已经在 context 中（session 开始时自动注入），Claude 看到已有 `preference_option.md`。

根据 prompt 指令 `"Do not write duplicate memories. First check if there is an existing memory you can update before writing a new one"`，Claude：

**Step 1**：FileEditTool 更新已有文件

```
file_path: ~/.claude/projects/<project>/memory/preference_option.md
old_string: |
  User prefers option A for feature X.
new_string: |
  User prefers option B for feature X.
  (Updated: previously preferred option A, changed per user request.)
```

**Step 2**：FileEditTool 更新索引

```
file_path: ~/.claude/projects/<project>/memory/MEMORY.md
old_string: "- [Option preference](preference_option.md) — user prefers option A"
new_string: "- [Option preference](preference_option.md) — user prefers option B"
```

**关键细节**：
- **无版本历史**：旧内容被直接文件编辑覆盖，不保留历史 diff
- **无自动扫描**：Claude 不会主动扫描全部记忆来找矛盾，它依赖 `MEMORY.md` 索引已在 context 中
- **如果遗漏了（写了两个文件）**：下次读取时，Memory Drift Caveat 兜底——发现矛盾则信任最新信息并清理旧的
- **frontmatter 也要更新**：prompt 要求 `"Keep the name, description, and type fields in memory files up-to-date with the content"`

#### 如果同一偏好涉及 team/private 冲突（feedback 场景）

假设场景变为：team 记忆有 `"项目统一用 Prettier 格式化"`，用户说 `"我这个文件不要用 Prettier"`。

Claude 在存 private feedback 之前，必须检查 team feedback。发现冲突后，两种处理：

**选项 A — 不存**：判断这是一次性需求，不值得记忆化。

**选项 B — 存但显式标注覆盖**：

```markdown
---
name: prettier exception
description: User overrides team Prettier convention for specific legacy file
type: feedback
---

User does not want Prettier formatting on legacy/config.js.
**Why:** File has hand-tuned alignment that Prettier would destroy.
**How to apply:** Skip Prettier for this specific file only.
**Note:** This overrides the team convention (team memory: "use Prettier on all JS files").
The team policy still applies to all other files.
```

### 1.6 Session Memory（会话记忆）

除了上面的持久化记忆（跨 session），Claude Code 还有 **Session Memory**——仅在当前会话内有效的笔记。

**源码位置**：`src/services/SessionMemory/`

机制完全不同：
- 固定模板结构（Session Title / Current State / Task specification / Files and Functions / Errors & Corrections / Learnings / Worklog 等 section）
- 由后台 subagent **定期自动提取**（不是用户触发），不阻塞主对话
- 每个 section 有 token 上限（`MAX_SECTION_LENGTH = 2000`），总量上限 12,000 tokens
- 更新方式是 **完整覆盖每个 section**（非追加），天然避免了同一 section 内的矛盾
- Session 结束后丢弃（不持久化）

Session Memory 没有矛盾处理的问题，因为它是整体覆盖式更新。

### 1.7 不该存的东西

源码中 `WHAT_NOT_TO_SAVE_SECTION` 明确排除：

- 代码模式、架构、文件路径、项目结构——读代码就知道
- Git 历史、最近变更、谁改了什么——`git log` / `git blame` 是权威源
- 调试方案、修复食谱——修复已经在代码里，commit message 有上下文
- 已在 CLAUDE.md 中记录的内容
- 临时任务细节：进行中的工作、临时状态、当前对话上下文

**即使用户明确要求存也不存**：
> `"These exclusions apply even when the user explicitly asks you to save. If they ask you to save a PR list or activity summary, ask what was *surprising* or *non-obvious* about it — that is the part worth keeping."`

### 1.8 记忆增长的规模问题：截断、召回与遗忘

一个核心问题是：**长期使用后记忆越来越多，MEMORY.md 索引也会越来越长。agent 还能正常工作吗？**

#### 1.8.1 硬截断机制

Claude Code 对 MEMORY.md 有**两道硬性上限**（源码 `src/memdir/memdir.ts`）：

```typescript
export const MAX_ENTRYPOINT_LINES = 200    // 索引最多加载 200 行
export const MAX_ENTRYPOINT_BYTES = 25_000  // 索引最多加载 25KB
```

每次 session 开始加载 MEMORY.md 时，会调用 `truncateEntrypointContent()` 函数。处理逻辑：

1. 如果行数超过 200 行 → 只取前 200 行（`contentLines.slice(0, MAX_ENTRYPOINT_LINES)`）
2. 如果截断后字节仍超过 25KB → 在 25KB 处的最后一个换行符截断（不切割半行）
3. 截断后在末尾追加一条 WARNING：

```
> WARNING: MEMORY.md is 312 lines (limit: 200). Only part of it was loaded.
> Keep index entries to one line under ~200 chars; move detail into topic files.
```

**这意味着：如果你的记忆索引超过 200 行，第 201 行及以后的索引条目不会出现在 agent 的初始上下文里。** 这些记忆文件还存在磁盘上，但 agent 开始工作时"看不见"它们的索引。

#### 1.8.2 三层召回漏斗

虽然 MEMORY.md 索引会被截断，但被截断的记忆**不是完全不可达**——Claude Code 有三层递进的召回机制：

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 1: MEMORY.md 索引自动加载（每次 session 自动触发）        │
│  • 自动注入 system prompt                                       │
│  • 覆盖范围：前 200 行 / 25KB                                   │
│  • 超出部分被截断丢弃                                           │
│  • 这是 agent 启动时的"初始视野"                                │
├─────────────────────────────────────────────────────────────────┤
│  Layer 2: Sonnet 侧查询主动召回（用户发消息时自动触发）          │
│  • 扫描记忆目录下所有 .md 文件（不是读 MEMORY.md 索引）         │
│  • 只读每个文件的 frontmatter（前 30 行，拿 name + description）│
│  • 把所有文件的摘要发给 Sonnet（便宜模型）问：                  │
│    "用户问了 X，这些记忆里哪些相关？选最多 5 个"                │
│  • 覆盖范围：最新 200 个 .md 文件（按修改时间倒序）             │
│  • 即使某条记忆的索引被 MEMORY.md 截断了，                      │
│    只要 .md 文件还在目录里，Sonnet 仍能找到它                   │
├─────────────────────────────────────────────────────────────────┤
│  Layer 3: Grep 搜索（agent 主动发起，非自动）                    │
│  • system prompt 教会 agent 用 grep 搜索记忆目录：              │
│    grep -rn "<term>" ~/.claude/.../memory/ --include="*.md"     │
│  • 覆盖范围：全部文件，无上限                                   │
│  • 但依赖 agent "意识到要去搜"以及关键词匹配质量                │
└─────────────────────────────────────────────────────────────────┘
```

**源码出处**：
- Layer 2（Sonnet 侧查询）：`src/memdir/findRelevantMemories.ts` → `findRelevantMemories()` 函数。扫描记忆文件由 `src/memdir/memoryScan.ts` → `scanMemoryFiles()` 实现，上限 `MAX_MEMORY_FILES = 200` 个文件
- Layer 3（Grep 搜索指引）：`src/memdir/memdir.ts` → `buildSearchingPastContextSection()` 函数，在 system prompt 中给出 grep 命令模板

#### 1.8.3 Layer 2 的工作细节

`findRelevantMemories()` 的完整流程：

1. `scanMemoryFiles()` 扫描记忆目录，读每个 `.md` 文件的 frontmatter（最多前 30 行），提取 `name`、`description`、`type`，按修改时间倒序排列，取最新 200 个
2. 构建一份"记忆清单"文本（每条一行：`[type] filename (timestamp): description`）
3. 用 `sideQuery()` 调用 Sonnet 模型，system prompt 为"你在为 Claude Code 选择有用的记忆"
4. Sonnet 返回最多 5 个文件名
5. 主模型读取这 5 个文件的完整内容

```typescript
// src/memdir/findRelevantMemories.ts 中的 Sonnet 选择 prompt
const SELECT_MEMORIES_SYSTEM_PROMPT = `You are selecting memories that will be
useful to Claude Code as it processes a user's query. ... Return a list of
filenames for the memories that will clearly be useful (up to 5). Only include
memories that you are certain will be helpful ...`
```

**关键细节**：如果 agent 最近正在使用某个工具（如 `mcp__X__spawn`），Sonnet 会跳过该工具的参考文档类记忆（因为对话里已经有了），但仍然会选中关于该工具的 warnings/gotchas 类记忆。

#### 1.8.4 实际效果：越老越容易被"遗忘"

三层漏斗的覆盖范围逐层递减：

| 层 | 机制 | 覆盖范围 | 触发方式 | 每次最多召回 |
|---|------|---------|---------|------------|
| 1 | MEMORY.md 索引 | 前 200 行 | 每次 session 自动 | 全部（200 行内） |
| 2 | Sonnet 侧查询 | 最新 200 个文件 | 用户发消息时自动 | 5 个文件 |
| 3 | Grep 搜索 | 全部文件 | agent 主动发起 | 取决于搜索结果 |

**实际效果：越老的记忆越容易被"遗忘"**——不是被删了，而是越来越难被触达。新索引条目排在 MEMORY.md 前面（或旧的被挤到 200 行之后），新文件在 Sonnet 扫描的 200 个文件里排序更高。这和人类记忆的衰减曲线其实很像。

#### 1.8.5 长期运行的 Dream/Distill 机制

在长期运行的 assistant 模式下（feature flag `KAIROS`），Claude Code 引入了一个额外的"记忆蒸馏"机制来缓解增长问题：

**日常**：agent 不直接修改 MEMORY.md，而是 **append-only** 写入每日日志文件：

```
~/.claude/projects/<slug>/memory/logs/2026/04/2026-04-24.md
```

**每晚**：一个 `/dream` skill 把累积的日志"蒸馏"成 topic files + 更新 MEMORY.md 索引。相当于定期整理压缩——把零散的日志条目合并、去重、更新到结构化的记忆文件里。

源码注释原文（`src/memdir/memdir.ts`）：
> "A separate nightly /dream skill distills these logs into topic files + MEMORY.md"

**但需注意**：`/dream` skill 的完整逻辑在当前源码中没有找到详细实现，可能还在开发中或作为外部 skill 部署。

#### 1.8.6 现实中没有自动清理

**重要事实**：Claude Code 目前**没有自动淘汰/过期/清理旧记忆的机制**。

- MEMORY.md 超过 200 行 → 只截断加载，不会帮你整理或删除
- 被截断的那些索引条目还在文件里，只是 agent 初始上下文看不到
- 没有 LRU 淘汰、没有过期时间、没有自动合并相似记忆
- Dream/Distill 是目前唯一的"整理"方案，但仅在 assistant 长期模式下可用

这本质上是**所有基于索引 + 文件读取的记忆方案的共同问题**——包括 WikiLLM 的 INDEX.md。当索引本身变得太大，不管是全量放进 context 还是由 agent 自己读取，都会遇到效率瓶颈。

---

## 二、Hermes Agent 记忆系统

### 2.1 内置记忆：MEMORY.md + USER.md

Hermes 的核心记忆系统比 Claude Code 简单得多——只有两个文件：

| 文件 | 用途 | 容量上限 | 典型条目数 |
|------|------|---------|-----------|
| `MEMORY.md` | Agent 的个人笔记（环境事实、约定、教训） | 2,200 字符（~800 tokens） | 8-15 条 |
| `USER.md` | 用户画像（偏好、沟通风格、期望） | 1,375 字符（~500 tokens） | 5-10 条 |

存储在 `~/.hermes/memories/`，每个 session 开始时作为**冻结快照**注入 system prompt（session 中间的修改会持久化到磁盘，但不会在当前 session 的 system prompt 中生效，下次 session 才可见）。

### 2.2 记忆操作工具

Hermes 通过 `memory` 工具操作记忆，支持三种 action：

| Action | 说明 | 矛盾相关 |
|--------|------|----------|
| `add` | 添加新条目 | ❌ 无矛盾检查 |
| `replace` | 用 `old_text` 子串匹配定位旧条目，替换为新内容 | ⚠️ 子串必须唯一匹配一条，否则报错 |
| `remove` | 用 `old_text` 子串匹配删除一条 | — |

**没有 `read` action**——记忆在 system prompt 中自动可见。

### 2.3 矛盾处理能力

#### 内置系统：极其有限

- **精确去重**：✅ 自动拒绝精确重复的条目
- **语义矛盾检测**：❌ 没有。如果存了"用户喜欢A"后又存"用户喜欢B"，两条会并存
- **陈旧标注**：❌ 没有时间戳或新鲜度标记
- **容量驱动的合并**：当 MEMORY.md 达到 2,200 字符上限时，`add` 会失败并返回错误，要求 agent 先 `replace` 合并旧条目或 `remove` 再 `add`。这是唯一会"被迫"清理旧记忆的时机

#### 外部记忆提供者插件

Hermes 的真正矛盾处理能力来自可选的**外部记忆提供者插件**（共 8 个，只能同时启用 1 个）。与矛盾处理相关的核心插件：

**Holographic（本地 SQLite + FTS5）**——唯一有显式矛盾检测的提供者：

```
fact_store 工具的 9 个 action：
- add       — 存储事实
- search    — 全文搜索
- probe     — 实体级代数召回（某个人/物的所有事实）
- related   — 关联事实查找
- reason    — 组合 AND 查询
- contradict — ✅ 自动检测已存储事实之间的逻辑矛盾
- update    — 更新事实
- remove    — 删除事实
- list      — 列出所有事实
```

配套机制：
- **Trust scoring**（信任评分）：每条事实有 0.0–1.0 的信任分（默认 0.5）
- **非对称反馈**：有用 +0.05，无用 -0.10——惩罚大于奖励，让低质量记忆逐渐淘汰

**Honcho（辩证式用户建模）**：

不是"矛盾检测"，而是"深层理解"。通过辩证法框架（dialectic reasoning）构建用户模型：
- 两层上下文注入：base layer（会话摘要 + 表征 + peer card）+ dialectic supplement（LLM 推理）
- 三个独立旋钮：`contextCadence`（base 刷新频率）、`dialecticCadence`（辩证 LLM 调用频率）、`dialecticDepth`（推理深度 1-3）
- 推理的是"用户为什么这样说"而非简单的"用户说了什么"

**Mem0（服务端 LLM 事实提取）**：

自动去重能力——Mem0 在服务端用 LLM 做事实提取，有 semantic dedup。但冲突检测能力不透明（取决于 Mem0 服务端实现）。

### 2.4 Memory Nudge（主动记忆推动）

Hermes 的核心差异化特性之一。Agent 会定期自我提醒，将重要信息从短期上下文持久化到长期记忆中——不需要用户触发。

但 nudge 流程是**单向写入**——它把值得记住的东西写进 MEMORY.md/USER.md，并不会在写入前做矛盾检查。

### 2.5 具体场景：用户偏好变更

**场景同上**：1h 前说喜欢 A，现在说喜欢 B。

Hermes 内置记忆的处理：

```python
# 1h 前
memory(action="add", target="user", content="User prefers option A")

# 现在——理想情况：agent 记得之前存过
memory(action="replace", target="user",
       old_text="option A",
       content="User prefers option B")

# 现在——遗漏情况：agent 忘了之前存过
memory(action="add", target="user", content="User prefers option B")
# → 结果：两条矛盾记忆并存于 USER.md，无自动检测
# → 直到容量满时才会被迫清理
```

---

## 三、OpenClaw 记忆系统

### 3.1 基于文件的被动记忆

OpenClaw 的"记忆"本质上就是**启动时注入的文本文件**：

| 文件 | 用途 |
|------|------|
| `AGENTS.md` | 操作指令 + "记忆"（实际上是人工维护的指导文件） |
| `SOUL.md` | 人格、边界、语气 |
| `USER.md` | 用户画像、称呼偏好 |
| `TOOLS.md` | 用户维护的工具使用笔记 |
| `IDENTITY.md` | Agent 名称/风格/emoji |
| `BOOTSTRAP.md` | 首次运行仪式（运行后删除） |

这些文件位于 `~/.openclaw/workspace/`（可配置），在每个新 session 的第一个 turn 注入到 agent context。

### 3.2 矛盾处理能力

**几乎为零**：

- ❌ 无记忆类型分类
- ❌ 无 frontmatter 元数据（无 type、description、时间戳）
- ❌ 无矛盾检测机制（写入时或读取时均无）
- ❌ 无陈旧标注
- ❌ 无容量管理（大文件被截断，但无合并/清理策略）
- ❌ 无去重
- ❌ 无 `replace`/`remove` 子串匹配工具——直接用通用文件编辑工具改

### 3.3 OpenClaw 的设计哲学

OpenClaw 有意选择了极简路线：
- 记忆就是文件，文件就是记忆
- 用户自己维护 `SOUL.md`、`AGENTS.md` 等文件
- Agent 可以用文件编辑工具（read/write/edit）修改这些文件，但没有专门的记忆工具
- 多 Agent 场景通过 workspace 隔离（每个 agent 有独立 workspace），而非记忆冲突检测

### 3.4 技能系统的冲突处理（非记忆）

值得一提的是，OpenClaw 对**同名 skill 冲突**有完整的优先级链：

```
<workspace>/skills (最高) → <workspace>/.agents/skills → ~/.agents/skills
→ ~/.openclaw/skills → bundled skills → skills.load.extraDirs (最低)
```

但这是技能文件的冲突解决，不是记忆内容的矛盾处理。

---

## 四、三者对比总结

### 4.1 矛盾处理能力对比

| 能力维度 | Claude Code | Hermes Agent（内置） | Hermes Agent（+插件） | OpenClaw |
|----------|------------|--------------------|--------------------|----------|
| 写入时去重 | ✅ "先查再写"指令 + 索引在 context | ✅ 精确重复拒绝 | ✅ Mem0 有 LLM 级去重 | ❌ |
| 写入时矛盾检查 | ✅ feedback 类型跨 scope 检查 | ❌ | ✅ Holographic 的 `contradict` | ❌ |
| 陈旧标注 | ✅ 超 1 天自动标注天数 | ❌ | ❌ | ❌ |
| 读取时漂移检测 | ✅ "当前观察 > 记忆" | ❌ | ⚠️ Holographic trust scoring | ❌ |
| 使用前验证 | ✅ grep/检查文件存在性 | ❌ | ❌ | ❌ |
| 容量驱动清理 | ✅ 索引 200 行 + 25KB 上限 | ✅ 2,200/1,375 字符上限 | 取决于提供者 | ❌ |
| 记忆类型分类 | ✅ 4 类 + scope | ❌ 2 个文件 | 取决于提供者 | ❌ |

### 4.2 记忆更新策略对比

| 策略 | Claude Code | Hermes Agent | OpenClaw |
|------|------------|-------------|----------|
| **更新方式** | FileEditTool 编辑 .md 文件 | `memory(action="replace")` 子串匹配替换 | 通用文件编辑工具 |
| **索引维护** | 需要同步更新 MEMORY.md 索引 | 无索引（内容直接在 system prompt） | 无索引 |
| **冲突解决** | 多层防御（5 层） | 容量挤压时替换 | 手动编辑 |
| **历史保留** | 无（覆盖写入） | 无（覆盖写入） | 无（覆盖写入） |
| **兜底机制** | Memory Drift Caveat | 无 | 无 |

### 4.3 设计理念对比

```
Claude Code:  防御性设计 —— "记忆是快照，不是真理。验证后再用。"
              五层防御，prompt-driven 矛盾处理，全靠 LLM 遵循指令执行

Hermes Agent: 进攻性设计 —— "越用越聪明，通过插件体系弥补核心不足"
              内置系统极简，矛盾处理能力外包给 8 种可选插件

OpenClaw:     极简设计 —— "记忆就是文件，用户自己管"
              不做记忆管理，也不做矛盾处理
```

---

## 五、核心发现

### 5.1 没有一个系统有"真正的"矛盾检测

三者都**没有**在系统层面实现自动化的语义矛盾检测（如"记忆 A 说用户喜欢深色，记忆 B 说用户喜欢浅色" → 自动标记冲突）。

- Claude Code 最接近——但它的矛盾检查是 **prompt 指令**（让 LLM 自己判断），不是代码逻辑
- Hermes 的 Holographic 插件有 `contradict` action——但这是一个**工具**（需要 agent 主动调用），不是自动触发
- 没有任何一个系统会在写入时自动扫描全部已有记忆来找语义冲突

### 5.2 Claude Code 的核心洞察

Claude Code 的记忆系统设计中最深刻的洞察是：**记忆是时间点观察，不是持久真理**。

```
"The memory says X exists" is not the same as "X exists now."
```

这一理念贯穿了整个系统——从 `memoryAge.ts` 的天数标注，到 `MEMORY_DRIFT_CAVEAT` 的漂移检测，到 `TRUSTING_RECALL_SECTION` 的使用前验证。它不试图保证记忆永远正确，而是**假设记忆可能已错，建立验证机制**。

### 5.3 Hermes 的策略选择

Hermes 选择了**核心极简 + 插件丰富**的路线。内置的 MEMORY.md/USER.md 极其简单（甚至没有矛盾检查），但通过 8 个外部记忆提供者插件覆盖了从知识图谱（Hindsight）到辩证推理（Honcho）到信任评分（Holographic）的完整谱系。这意味着矛盾处理能力取决于用户选择启用了哪个插件。

### 5.4 一个开放问题

三者都没有解决的问题：**跨 session 的隐式偏好漂移**。

假设用户在 20 次会话中，前 10 次倾向方案 A，后 10 次逐渐转向方案 B，但从未明确说过"我不再喜欢 A 了"。所有三个系统都不会自动检测到这种趋势——需要用户显式表达偏好变更，或者 agent 恰好在某次对话中回顾旧记忆并发现不一致。

这可能是下一代记忆系统需要解决的核心挑战。
