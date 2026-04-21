# Claude Code Sub-Agent 机制深度解析

> 撰写日期：2026年4月21日
> 基于 Claude Code 源码分析

---

## 一、概述

Claude Code 的 Sub-Agent 机制是其**多层委托执行架构**的核心，允许主 Agent 将复杂的子任务分发给独立的子 Agent 执行。这套系统包含三个层级——从最轻量的单次 API 调用，到完整的带工具执行的 Agent 循环，再到模型主动发起的并行 Agent 编排。

```
┌──────────────────────────────────────────────────────┐
│                 用户交互层                             │
│                QueryEngine                            │
└──────────────────────┬───────────────────────────────┘
                       │
┌──────────────────────▼───────────────────────────────┐
│              主 Agent 循环 (queryLoop)                 │
│          while(true) { call API → 执行工具 }          │
│                                                       │
│   ┌─────────────┐  ┌──────────────┐  ┌────────────┐ │
│   │  sideQuery   │  │ forkedAgent  │  │  AgentTool  │ │
│   │ (单次调用)   │  │ (缓存共享)   │  │ (模型发起)  │ │
│   └─────────────┘  └──────────────┘  └──────┬─────┘ │
│                                              │       │
│                    ┌─────────────────────────┤       │
│                    ▼                         ▼       │
│             ┌──────────┐            ┌──────────────┐ │
│             │ Fork 模式 │            │ Agent 类型    │ │
│             │ (继承上下文)│           │ (独立启动)    │ │
│             └──────────┘            └──────────────┘ │
│                                                       │
│   ┌──────────────────────────────────────────────┐   │
│   │           Coordinator 模式                    │   │
│   │    (协调者 + 多个 Worker 的任务编排)          │   │
│   └──────────────────────────────────────────────┘   │
└───────────────────────────────────────────────────────┘
```

---

## 二、三层调用机制

### 第一层：sideQuery — 最轻量级

**位置**：`src/utils/sideQuery.ts`

sideQuery 是最简单的子 Agent 模式——**单次 API 调用，无工具执行，无多轮对话**。

```
主 Agent → sideQuery(prompt) → Claude API → 返回文本答案
           不走 queryLoop
           不执行工具
           不追踪 usage
           不记录 transcript
           默认 max_tokens: 1024
```

**使用场景**：
- 记忆召回时的语义选择（用 Sonnet 判断哪条记忆相关）
- 权限分类（判断操作是否安全）
- Session 搜索（搜索历史对话的关键词提取）
- 任何"问模型一个问题，拿一个答案"的场景

**特点**：极低开销，不产生子 Agent ID，不参与任务管理系统。

### 第二层：runForkedAgent — 带 Prompt Cache 共享的完整循环

**位置**：`src/utils/forkedAgent.ts`

forkedAgent 启动**与主 Agent 完全相同的 `query()` 函数**，拥有完整的工具执行能力和多轮对话能力，但通过精心设计的状态隔离确保不干扰主 Agent。

**核心优势：Prompt Cache 共享**

```
主 Agent 的 API 请求:
  [system_prompt][消息历史][当前turn]
         ↓ 这部分完全相同 ↓
Fork 子 Agent 的 API 请求:
  [system_prompt][消息历史][fork指令]
  
→ ~80% 成本节省（90% 命中 cache_read）
```

实现这个缓存共享需要**字节级一致性**（`CacheSafeParams`）：
- clone `readFileState`，不修改原始状态
- 不过滤 incomplete tool calls
- 使用统一的 placeholder 文本
- 禁止随意设置 `maxOutputTokens`

**createSubagentContext() — 状态隔离工厂**：

| 状态 | 策略 | 原因 |
|------|------|------|
| `readFileState` | Clone | 子 Agent 读文件不影响父的缓存 |
| `contentReplacementState` | Clone | 工具结果替换独立 |
| `setAppState` | No-op | 子 Agent 不修改全局 UI 状态 |
| `setAppStateForTasks` | 穿透到根 | 任务注册/进度必须全局可见 |
| `abort` | 单向传播 | 父 abort → 子 abort，子不影响父 |

**使用场景**：
- Session Memory 提取（后台 fork 分析对话并增量更新笔记）
- 自动记忆提取（fire-and-forget，分析对话提取知识）
- Full Compact 摘要生成（长上下文压缩）

### 第三层：AgentTool — 模型主动启动的子 Agent

**位置**：`src/tools/AgentTool/AgentTool.tsx` + `src/tools/AgentTool/runAgent.ts`

这是**最核心也最复杂**的层级。它是一个标准的 Claude Code Tool，模型可以在对话中主动调用它来启动子 Agent。

**输入参数**：

```typescript
{
  prompt: string,              // 子 Agent 要执行的任务
  description: string,         // 3-5 词的简短描述
  subagent_type?: string,      // Agent 类型（省略则走 Fork 路径）
  model?: 'sonnet'|'opus'|'haiku', // 模型覆盖
  run_in_background?: boolean, // 是否后台运行
  isolation?: 'worktree'|'remote', // 隔离模式
  name?: string,               // 命名（用于 SendMessage 寻址）
  cwd?: string,                // 工作目录覆盖
}
```

**执行生命周期**：

```
1. 解析 Agent 定义 → 找到对应的 AgentDefinition
2. 初始化 Agent 专属 MCP 服务器
3. 解析工具集（根据 tools/disallowedTools 过滤）
4. 获取模型（agent.model → 父模型 → 显式 model 参数）
5. 渲染系统提示词
6. createSubagentContext() 创建隔离上下文
7. 可选：执行 SubagentStart hooks
8. 运行 query()（完整的 Agent 循环）
9. finally 清理：MCP 连接、bash 进程、transcript、
   file state、hooks、agent tracking、todos
```

---

## 三、内置 Agent 类型

Claude Code 内置了 6 种 Agent 类型，各有明确分工：

### 1. General Purpose（通用 Agent）

```typescript
agentType: 'general-purpose'
tools: ['*']           // 所有工具
model: 默认 subagent model
```

**定位**：通用万能 Agent，用于"搜索代码、调研复杂问题、执行多步骤任务"。当不确定该用什么 Agent 时的默认选择。

**System Prompt 核心**：
> "Given the user's message, you should use the tools available to complete the task. Complete the task fully — don't gold-plate, but don't leave it half-done."

### 2. Explore（探索 Agent）

```typescript
agentType: 'Explore'
disallowedTools: [Agent, FileEdit, FileWrite, NotebookEdit, ExitPlanMode]
model: 'haiku'（外部用户）/ 'inherit'（内部）
omitClaudeMd: true     // 省略 CLAUDE.md 节省 token
```

**定位**：**只读**的代码搜索专家。快速查找文件、搜索关键词、回答代码库相关问题。

**关键限制**：
- 严格禁止创建、修改、删除任何文件
- 不能启动子 Agent（防止递归）
- 使用 Haiku 模型以获得最快速度
- 省略 CLAUDE.md 节省 ~5-15 Gtok/week（34M+ 次 Explore 调用）

**并行优化**：
> "Wherever possible you should try to spawn multiple parallel tool calls for grepping and reading files"

这是使用频率最高的内置 Agent——每周被调用 3400 万+次。

### 3. Plan（规划 Agent）

```typescript
agentType: 'Plan'
disallowedTools: [Agent, FileEdit, FileWrite, NotebookEdit, ExitPlanMode]
model: 'inherit'
omitClaudeMd: true
```

**定位**：软件架构师，专门设计实现方案。输出包含分步实现策略、依赖排序和关键文件列表。

**要求输出格式**：必须以"Critical Files for Implementation"列表结尾，列出 3-5 个最关键的文件。

### 4. Fork（分叉 Agent）

```typescript
agentType: 'fork'
tools: ['*']
model: 'inherit'
permissionMode: 'bubble'  // 权限请求冒泡到父终端
```

**定位**：继承父 Agent 完整上下文的"分身"。当省略 `subagent_type` 时触发。

**独特机制**：
- **继承父的完整消息历史和系统提示词**（其他 Agent 都从零开始）
- **与父共享 Prompt Cache**（字节级一致）
- **权限冒泡**：权限请求显示在父 Agent 的终端上
- **反递归保护**：Fork 子消息包含 `<fork-boilerplate>` 标签，`isInForkChild()` 检测到则拒绝再次 fork

**Fork 子 Agent 的消息构建**（`buildForkedMessages()`）：

```
所有 Fork 共享的前缀（命中缓存）:
  [父的完整 assistant message（所有 tool_use 块）]
  [为每个 tool_use 填充相同的 placeholder result]

每个 Fork 独有的部分（唯一不同）:
  [<fork-boilerplate>指令 + 具体 directive]
```

**Fork Boilerplate 核心规则**：
```
STOP. READ THIS FIRST.
You are a forked worker process. You are NOT the main agent.
1. Do NOT spawn sub-agents; execute directly.
2. Do NOT converse, ask questions, or suggest next steps
3. USE your tools directly: Bash, Read, Write, etc.
4. Stay strictly within your directive's scope.
5. Keep your report under 500 words.
6. Your response MUST begin with "Scope:".
```

### 5. Verification（验证 Agent）

```typescript
agentType: 'verification'
disallowedTools: [Agent, FileEdit, FileWrite, NotebookEdit, ExitPlanMode]
model: 'inherit'
color: 'red'
background: true       // 默认后台运行
```

**定位**：对抗性测试专家。它的工作**不是确认实现正确，而是尝试破坏它**。

**System Prompt 核心哲学**：
> "Your job is not to confirm the implementation works — it's to try to break it."
> "You have two documented failure patterns: verification avoidance, and being seduced by the first 80%."

**自我防欺骗机制**：
> "You will feel the urge to skip checks. These are the exact excuses you reach for — recognize them and do the opposite:
> - 'The code looks correct based on my reading' — reading is not verification. Run it.
> - 'The implementer's tests already pass' — the implementer is an LLM. Verify independently.
> - 'This is probably fine' — probably is not verified. Run it."

**必须输出格式**：每个检查都要有 Command run、Output observed、Result，最终以 `VERDICT: PASS/FAIL/PARTIAL` 结束。

### 6. Claude Code Guide

内置的帮助向导 Agent，用于回答关于 Claude Code 本身的使用问题。仅在非 SDK 入口点时可用。

---

## 四、Coordinator 模式 — 高级并行编排

**位置**：`src/coordinator/coordinatorMode.ts`

Coordinator 模式是一种**与 Fork 互斥**的高级并行策略。启用后，主 Agent 变成纯粹的"协调者"，不直接执行任务，而是管理多个 Worker Agent。

### 架构

```
┌─────────────────────────────────────────┐
│              Coordinator                 │
│  （不直接使用工具，只做任务分配和综合）  │
│                                          │
│  工具：Agent()、SendMessage()、TaskStop() │
└──────┬──────────┬──────────┬────────────┘
       │          │          │
  ┌────▼────┐ ┌──▼────┐ ┌──▼──────┐
  │ Worker A │ │Worker B│ │Worker C │
  │ (研究)   │ │(实现)  │ │(验证)   │
  └─────────┘ └───────┘ └─────────┘
       │          │          │
       ▼          ▼          ▼
  <task-notification> 消息返回给 Coordinator
```

### Coordinator 的核心 Prompt

```
You are a **coordinator**. Your job is to:
- Help the user achieve their goal
- Direct workers to research, implement and verify code changes
- Synthesize results and communicate with the user
- Answer questions directly when possible
```

### 四阶段工作流

| 阶段 | 执行者 | 目的 |
|------|--------|------|
| **Research** | Workers（并行） | 调查代码库，理解问题 |
| **Synthesis** | **Coordinator 自己** | 理解研究结果，设计实现方案 |
| **Implementation** | Workers | 按方案执行修改，提交 |
| **Verification** | Workers | 测试变更是否正确 |

### Worker 通信机制

Worker 完成后通过 `<task-notification>` XML 消息通知 Coordinator：

```xml
<task-notification>
  <task-id>{agentId}</task-id>
  <status>completed|failed|killed</status>
  <summary>{人类可读的状态摘要}</summary>
  <result>{agent 的最终文本响应}</result>
  <usage>
    <total_tokens>N</total_tokens>
    <tool_uses>N</tool_uses>
    <duration_ms>N</duration_ms>
  </usage>
</task-notification>
```

### 并发策略

Coordinator Prompt 中明确强调：
> "**Parallelism is your superpower.** Workers are async. Launch independent workers concurrently whenever possible."

具体规则：
- **只读任务**（研究）→ 自由并行
- **写操作任务**（实现）→ 同一文件区域一次一个
- **验证**→ 有时可与实现并行（不同文件区域）

### "永不委托理解" 原则

Coordinator 最重要的规则：

```
// 反模式（坏）
Agent({ prompt: "Based on your findings, fix the auth bug" })

// 正确模式（好）
Agent({ prompt: "Fix the null pointer in src/auth/validate.ts:42. 
  The user field on Session (src/auth/types.ts:15) is undefined when 
  sessions expire but the token remains cached. Add a null check 
  before user.id access — if null, return 401 with 'Session expired'." })
```

---

## 五、Fork vs Coordinator 对比

| 维度 | Fork | Coordinator |
|------|------|-------------|
| **触发方式** | 模型省略 `subagent_type` | 环境变量 `CLAUDE_CODE_COORDINATOR_MODE` |
| **上下文** | 完整继承父的消息历史 | Worker 从零开始（prompt 必须自包含） |
| **Prompt Cache** | 与父共享（字节级一致） | 不共享 |
| **权限处理** | bubble 到父终端 | 各 Worker 独立处理 |
| **通信** | 无（各自独立执行后返回） | `SendMessage` 可继续已有 Worker |
| **适用场景** | "帮我做这个子任务"（对话分支） | "把大任务拆分给多个工人"（任务编排） |
| **互斥** | 与 Coordinator 互斥 | 与 Fork 互斥 |

---

## 六、权限模型

Sub-Agent 的权限通过 `permissionMode` 控制：

| Mode | 行为 | 典型使用 |
|------|------|---------|
| `default` | 弹确认框问用户 | 普通前台 Agent |
| `plan` | 只能规划，执行需审批 | 需要控制的 teammate |
| `acceptEdits` | 自动接受编辑 | 信任的自动化 Agent |
| `bypassPermissions` | 跳过所有权限检查 | 完全信任的环境 |
| `dontAsk` | 不问、直接拒绝 | 后台 Agent（不能弹框） |
| `bubble` | 权限请求冒泡到父终端 | **Fork 专用** |
| `auto` | YOLO 分类器驱动 | 自动判断是否安全 |

**关键设计**：后台 Agent（`isAsync=true`）默认设置 `shouldAvoidPermissionPrompts: true`，防止死锁——后台 Agent 没有终端可以弹框。Fork 的 `bubble` 模式是特殊解法——它将权限请求冒泡到父 Agent 的终端上显示。

---

## 七、工具过滤机制

不同类型的 Agent 能使用的工具不同，通过多层过滤实现：

### 全局禁止（所有 Agent）

```typescript
ALL_AGENT_DISALLOWED_TOOLS = {
  // 防止 Agent 修改自己的配置或启动新会话等
}
```

### 自定义 Agent 额外禁止

```typescript
CUSTOM_AGENT_DISALLOWED_TOOLS = {
  // 用户定义的 Agent 额外不能使用的工具
}
```

### 异步 Agent 白名单

```typescript
ASYNC_AGENT_ALLOWED_TOOLS = {
  // 后台运行的 Agent 只能使用这些工具
}
```

### Agent 定义级过滤

每个 Agent 可以指定：
- `tools: ['*']` — 所有工具（通配符）
- `tools: ['Bash', 'Read', 'Grep']` — 明确列出
- `disallowedTools: ['FileEdit', 'FileWrite']` — 排除列表

最终工具集 = `(全局可用 - 全局禁止 - 自定义禁止 - 异步限制) ∩ Agent 定义的 tools - Agent 定义的 disallowedTools`

---

## 八、Agent MCP 服务器

Agent 可以定义自己的 MCP 服务器（在 agent frontmatter 中配置），这些服务器**叠加**到父的 MCP 客户端之上：

```typescript
// Agent 定义中
mcpServers: [
  "existing-server-name",    // 引用已有的（共享，不清理）
  { "my-server": { ... } }  // 内联定义（Agent 独有，结束后清理）
]
```

**生命周期**：
- 引用已有服务器 → 共享父的连接，不清理
- 内联定义新服务器 → Agent 启动时连接，结束后清理
- 受 `pluginOnlyPolicy` 限制：用户定义的 Agent 在严格模式下不能使用 frontmatter MCP

---

## 九、Worktree 隔离

设置 `isolation: "worktree"` 时，Agent 在**临时 git worktree** 中工作：

```
主仓库:  /project/main/
  │
  └── Agent Worktree: /project/.git/worktrees/agent-xxx/
      （独立的文件系统副本，可以自由修改）
```

**工作流**：
1. `createAgentWorktree()` 从当前 HEAD 创建临时 worktree
2. Agent 在 worktree 中自由修改文件
3. 完成后 `hasWorktreeChanges()` 检查是否有变更
4. 有变更 → 保留 worktree，返回路径和分支名给父 Agent
5. 无变更 → `removeAgentWorktree()` 自动清理

**用途**：让 Agent 大胆尝试修改而不污染主分支。

---

## 十、后台 Agent 与任务系统

### 任务类型

```typescript
type TaskType = 
  | 'local_bash'           // 本地 Bash 命令
  | 'local_agent'          // 本地子 Agent
  | 'remote_agent'         // 远程 Agent
  | 'in_process_teammate'  // 进程内协作者
  | 'local_workflow'       // 本地工作流
  | 'monitor_mcp'          // MCP 监控
  | 'dream'                // 后台推演

type TaskStatus = 'pending' | 'running' | 'completed' | 'failed' | 'killed'
```

### 后台执行流程

```
1. Agent 设置 run_in_background: true
2. AgentTool 立即返回 { status: 'async_launched', agentId, outputFile }
3. 主 Agent 继续处理其他任务
4. 后台 Agent 独立执行，进度写入 outputFile
5. 完成后通过 <task-notification> 通知主 Agent
6. 主 Agent 在下一个 turn 收到通知
```

### Auto-Background 机制

```typescript
function getAutoBackgroundMs(): number {
  // 启用后，Agent 运行超过 120 秒自动转入后台
  if (env.CLAUDE_AUTO_BACKGROUND_TASKS || growthbook.tengu_auto_background_agents) {
    return 120_000;
  }
  return 0;
}
```

---

## 十一、性能优化设计

### 1. Prompt Cache 最大化

Fork 模式的设计核心就是 cache 共享。`buildForkedMessages()` 确保多个 fork 从同一点分叉时，**99% 的 API 请求前缀完全相同**：

```
共享部分（命中缓存）:
  system_prompt + 历史消息 + 父 assistant message + placeholder tool results

不同部分（仅此不同）:
  每个 fork 的具体 directive 文本
```

### 2. Agent 列表从 Tool 描述移到附件消息

Agent 列表（所有可用 Agent 类型和描述）曾经嵌入在 AgentTool 的 tool description 中。但 MCP 连接变化、插件重载等会导致列表变化 → description 变化 → **全量 tool schema 缓存失效**。

优化后（`shouldInjectAgentListInMessages()`）：Agent 列表作为附件消息注入，tool description 保持静态。**节省 ~10.2% 的 cache_creation token**。

### 3. Explore/Plan 省略 CLAUDE.md

Explore 和 Plan 是只读 Agent，不需要 CLAUDE.md 中的 commit/PR/lint 规则。`omitClaudeMd: true` 跳过这些内容，**每周节省 ~5-15 Gtok**。

### 4. One-Shot Agent 省略 SendMessage 提示

```typescript
ONE_SHOT_BUILTIN_AGENT_TYPES = new Set(['Explore', 'Plan'])
```

Explore 和 Plan 执行一次就返回，不需要 `SendMessage` 继续。省略 agentId/SendMessage/usage 信息，**每次节省 ~135 字符 × 34M Explore/week**。

---

## 十二、Agent Context 隔离（AsyncLocalStorage）

**位置**：`src/utils/agentContext.ts`

当多个 Agent 后台并发运行时，如何确保它们的上下文不相互干扰？Claude Code 使用 Node.js 的 `AsyncLocalStorage` 实现异步执行链隔离：

```typescript
const agentContextStorage = new AsyncLocalStorage<AgentContext>()

// 在 Agent 执行链中运行
function runWithAgentContext<T>(context: AgentContext, fn: () => T): T {
  return agentContextStorage.run(context, fn)
}

// 在任意位置获取当前 Agent 上下文
function getAgentContext(): AgentContext | undefined {
  return agentContextStorage.getStore()
}
```

**为什么不用 AppState？**

> When agents are backgrounded (ctrl+b), multiple agents can run concurrently in the same process. AppState is a single shared state that would be overwritten, causing Agent A's events to incorrectly use Agent B's context. AsyncLocalStorage isolates each async execution chain, so concurrent agents don't interfere with each other.

两种 Agent 上下文类型：
- `SubagentContext`：普通子 Agent（agentType: 'subagent'）
- `TeammateAgentContext`：多 Agent 协作中的团队成员（agentType: 'teammate'）

---

## 十三、总结

Claude Code 的 Sub-Agent 机制是一个**精心分层、性能优先、安全隔离**的系统：

| 层级 | 机制 | 成本 | 上下文 | 工具 | 典型场景 |
|------|------|------|--------|------|---------|
| **L1** | sideQuery | 极低 | 无状态 | 无 | 语义判断、分类 |
| **L2** | forkedAgent | 中等（cache 共享） | 继承 | 完整 | 记忆提取、摘要生成 |
| **L3** | AgentTool | 可变 | 独立/继承 | 按类型过滤 | 代码搜索、实现、验证 |
| **L4** | Coordinator | 高 | 独立 + 通信 | Worker 受限 | 大型多步任务编排 |

**设计哲学**：
1. **渐进式复杂度**：简单任务用轻量方案，复杂任务才启用完整机制
2. **缓存为王**：Fork 模式、静态 tool description、省略 CLAUDE.md——所有优化都指向减少 cache miss
3. **安全隔离**：状态 clone、单向 abort 传播、权限冒泡、worktree 隔离
4. **反递归保护**：Fork 子 Agent 不能再 fork，Explore/Plan 不能启动子 Agent
5. **模型也是用户**：Verification Agent 的 prompt 设计体现了对"LLM 会偷懒"的深刻理解
