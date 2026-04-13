# Claude Code 核心机制深度剖析：Agent Loop · 上下文管理 · 工具系统

> 基于源码逐行分析，聚焦三大核心系统的**实现细节**。

---

## 目录

1. [Agent Loop：一个 while(true) 如何驱动整个 Agent](#1-agent-loop)
   - 1.1 [queryLoop 全貌：1750 行的 while(true)](#11-queryloop-全貌)
   - 1.2 [Loop State：跨迭代的可变状态](#12-loop-state)
   - 1.3 [单次迭代的完整流程](#13-单次迭代的完整流程)
   - 1.4 [流式工具并行：模型还在说话就开始干活](#14-流式工具并行)
   - 1.5 [异常恢复：7 种 continue 路径](#15-异常恢复)
   - 1.6 [Stop Hooks：Turn 结束后的生命周期](#16-stop-hooks)
   - 1.7 [Token Budget：自动续写机制](#17-token-budget)
2. [上下文管理：四把手术刀 + 一个兜底](#2-上下文管理)
   - 2.1 [总体策略：按顺序依次执行](#21-总体策略)
   - 2.2 [Tool Result Budget：聚合级别的预算裁切](#22-tool-result-budget)
   - 2.3 [HISTORY_SNIP：最精细的一层](#23-history_snip)
   - 2.4 [Microcompact：缓存层面的手术](#24-microcompact)
   - 2.5 [CONTEXT_COLLAPSE：结构化归档](#25-context_collapse)
   - 2.6 [Autocompact：最后的兜底](#26-autocompact)
   - 2.7 [Reactive Compact：等 API 报错再动手](#27-reactive-compact)
   - 2.8 [压缩流水线的完整代码路径](#28-压缩流水线的完整代码路径)
3. [工具系统：40+ 工具，零继承的函数式架构](#3-工具系统)
   - 3.1 [Tool 接口：一切皆 buildTool()](#31-tool-接口)
   - 3.2 [工具注册：三阶段动态组装](#32-工具注册)
   - 3.3 [工具执行：从权限到结果的完整链路](#33-工具执行)
   - 3.4 [并发控制：StreamingToolExecutor 详解](#34-并发控制)
   - 3.5 [BashTool：最复杂的单个工具（18 个文件）](#35-bashtool)
   - 3.6 [FileEditTool：原子性读-改-写](#36-fileedittool)
   - 3.7 [工具结果如何回流到 Agent Loop](#37-工具结果回流)
4. [记忆系统：从 MEMORY.md 到自动提取的完整链路](#4-记忆系统)
   - 4.1 [MEMORY.md vs CLAUDE.md](#41-记忆-vs-指令)
   - 4.2 [记忆目录结构](#42-记忆目录结构)
   - 4.3 [记忆文件格式（四类型分类法）](#43-记忆文件格式)
   - 4.4 [MEMORY.md 索引文件](#44-memorymd-索引文件)
   - 4.5 [记忆注入：三条路径](#45-记忆注入三个时机三条路径)
   - 4.6 [自动记忆提取：后台子 Agent](#46-自动记忆提取后台子-agent)
   - 4.7 [记忆扫描与目录管理](#47-记忆扫描与目录管理)
   - 4.8 [团队记忆 (TEAMMEM)](#48-团队记忆-teammem)
   - 4.9 [KAIROS 模式（长驻助手）](#49-kairos-模式长驻助手)
   - 4.10 [/memory 命令](#410-memory-命令)
   - 4.11 [记忆与 Compact 的交互](#411-记忆与-compact-的交互)
   - 4.12 [关键常量速查](#412-记忆系统关键常量)
   - 4.13 [记忆系统完整数据流](#413-记忆系统完整数据流)

---

## 1. Agent Loop

### 1.1 queryLoop 全貌

整个 Claude Code 最核心的文件是 `src/query.ts`，~1750 行。注意不是 `QueryEngine.ts`（那个是外层的会话管理器，负责注入 system prompt、管理消息历史、处理用户输入等），**真正的 Agent 大脑**在 `query.ts` 里的 `queryLoop` 函数，一个 `while (true)` 循环。

```typescript
// src/query.ts - 简化后的骨架
async function* queryLoop(
  params: QueryParams,
  consumedCommandUuids: string[],
): AsyncGenerator<StreamEvent | Message | ToolUseSummaryMessage, Terminal> {

  // 不可变参数 — 整个循环期间不会被重新赋值
  const { systemPrompt, userContext, systemContext, canUseTool, 
          fallbackModel, querySource, maxTurns, skipCacheWrite } = params

  // 可变的跨迭代状态
  let state: State = {
    messages: params.messages,
    toolUseContext: params.toolUseContext,
    autoCompactTracking: undefined,
    maxOutputTokensRecoveryCount: 0,
    hasAttemptedReactiveCompact: false,
    maxOutputTokensOverride: params.maxOutputTokensOverride,
    pendingToolUseSummary: undefined,
    stopHookActive: undefined,
    turnCount: 1,
    transition: undefined,  // 上一次迭代为什么 continue
  }

  // 一次性快照：运行时 feature gate 和环境配置
  const config = buildQueryConfig()
  
  // 记忆预取（using 语法确保 generator 退出时自动 dispose）
  using pendingMemoryPrefetch = startRelevantMemoryPrefetch(
    state.messages, state.toolUseContext
  )

  while (true) {
    // ... 核心循环体（下文逐段剖析）
  }
}
```

**返回类型**是 `AsyncGenerator`：`queryLoop` 是一个异步生成器函数，通过 `yield` 向外流式发射消息（assistant 文本、工具结果、progress 事件、system 消息），调用方（`QueryEngine`/REPL）逐个消费并渲染。

**`query()` 是 `queryLoop` 的薄包装**：唯一的额外逻辑是在正常返回时通知已消费的命令 `'completed'`：

```typescript
export async function* query(params: QueryParams) {
  const consumedCommandUuids: string[] = []
  const terminal = yield* queryLoop(params, consumedCommandUuids)
  for (const uuid of consumedCommandUuids) {
    notifyCommandLifecycle(uuid, 'completed')
  }
  return terminal
}
```

### 1.2 Loop State

跨迭代状态被集中在一个 `State` 对象中，每次 `continue` 时整体替换（不是 9 个独立赋值）：

```typescript
type State = {
  messages: Message[]                   // 当前消息序列
  toolUseContext: ToolUseContext         // 工具执行上下文
  autoCompactTracking: AutoCompactTrackingState | undefined  // 压缩追踪
  maxOutputTokensRecoveryCount: number  // max_output_tokens 恢复计数
  hasAttemptedReactiveCompact: boolean  // 是否已尝试响应式压缩
  maxOutputTokensOverride: number | undefined  // token 上限覆盖
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined
  stopHookActive: boolean | undefined   // stop hook 是否激活
  turnCount: number                     // 当前 turn 计数
  transition: Continue | undefined      // 上一次 continue 的原因
}
```

`transition` 字段记录了**为什么**上一次迭代做了 `continue` 而非退出。这用于避免恢复死循环（例如 collapse drain 后重试仍然 413，就不再走 drain 路径了）。

### 1.3 单次迭代的完整流程

每次 `while(true)` 迭代的执行步骤：

```
┌─── 迭代开始 ────────────────────────────────────────────────┐
│                                                              │
│  1. 解构 state, 启动 Skill 发现预取                          │
│  2. TOOL RESULT BUDGET — 对累计工具结果做聚合级裁切            │
│  3. HISTORY_SNIP — 删除噪声消息（Feature-gated）              │
│  4. MICROCOMPACT — 缓存层 token 删除                         │
│  5. CONTEXT_COLLAPSE — 结构化归档旧轮次（Feature-gated）       │
│  6. AUTOCOMPACT — 超阈值时摘要压缩整个历史                    │
│  7. 阻塞限制检查 — 预防 prompt-too-long                      │
│                                                              │
│  ════════════ Claude API 流式调用 ════════════                │
│                                                              │
│  8. 流式接收消息                                              │
│     ├─ assistant 文本 → yield 给调用者                        │
│     ├─ tool_use block → 加入 toolUseBlocks + 流式执行器       │
│     ├─ 可恢复错误 (413/max_tokens/media) → 暂扣不 yield      │
│     └─ 模型回退 → 清空并重试                                 │
│                                                              │
│  9. Post-sampling hooks（异步触发）                           │
│ 10. 中断检查 — 用户按了 Ctrl+C？                              │
│ 11. 上一轮的 ToolUseSummary（Haiku 生成的摘要）              │
│                                                              │
│  ─── 如果没有工具调用 (needsFollowUp=false) ───               │
│     12. 暂扣错误恢复（collapse drain / reactive compact）     │
│     13. max_output_tokens 恢复 / 升级                        │
│     14. Stop hooks                                            │
│     15. Token budget 续写检查                                 │
│     16. return { reason: 'completed' }                        │
│                                                              │
│  ─── 如果有工具调用 (needsFollowUp=true) ───                  │
│     17. 工具执行（流式 or 批量）                               │
│     18. ToolUseSummary 异步生成（Haiku）                      │
│     19. 附件注入（文件变更、排队命令、记忆、技能）              │
│     20. maxTurns 检查                                         │
│     21. state = next → continue（回到步骤1）                  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

让我们看几个最关键的环节的实际代码。

#### 步骤 2-6：压缩流水线

这五步按顺序执行，**前面能搞定就不触发后面**：

```typescript
// ① Tool Result Budget — 对过大的工具结果裁切到磁盘
messagesForQuery = await applyToolResultBudget(
  messagesForQuery,
  toolUseContext.contentReplacementState,
  persistReplacements ? records => void recordContentReplacement(records, agentId) : undefined,
  new Set(tools.filter(t => !Number.isFinite(t.maxResultSizeChars)).map(t => t.name)),
)

// ② HISTORY_SNIP — 删除噪声消息
if (feature('HISTORY_SNIP')) {
  const snipResult = snipModule!.snipCompactIfNeeded(messagesForQuery)
  messagesForQuery = snipResult.messages
  snipTokensFreed = snipResult.tokensFreed
}

// ③ MICROCOMPACT — 缓存层 token 删除
const microcompactResult = await deps.microcompact(messagesForQuery, toolUseContext, querySource)
messagesForQuery = microcompactResult.messages

// ④ CONTEXT_COLLAPSE — 结构化归档
if (feature('CONTEXT_COLLAPSE') && contextCollapse) {
  const collapseResult = await contextCollapse.applyCollapsesIfNeeded(
    messagesForQuery, toolUseContext, querySource
  )
  messagesForQuery = collapseResult.messages
}

// ⑤ AUTOCOMPACT — 全量摘要压缩
const { compactionResult, consecutiveFailures } = await deps.autocompact(
  messagesForQuery, toolUseContext,
  { systemPrompt, userContext, systemContext, toolUseContext, forkContextMessages: messagesForQuery },
  querySource, tracking, snipTokensFreed,
)
```

#### 步骤 8：流式接收 + 流式工具执行

```typescript
for await (const message of deps.callModel({
  messages: prependUserContext(messagesForQuery, userContext),
  systemPrompt: fullSystemPrompt,
  tools: toolUseContext.options.tools,
  signal: toolUseContext.abortController.signal,
  // ... 其他参数
})) {
  // 暂扣可恢复错误（不 yield，留给后续恢复逻辑）
  let withheld = false
  if (reactiveCompact?.isWithheldPromptTooLong(message)) withheld = true
  if (isWithheldMaxOutputTokens(message)) withheld = true
  if (!withheld) yield yieldMessage

  if (message.type === 'assistant') {
    assistantMessages.push(message)
    const msgToolUseBlocks = message.message.content
      .filter(c => c.type === 'tool_use') as ToolUseBlock[]
    
    if (msgToolUseBlocks.length > 0) {
      toolUseBlocks.push(...msgToolUseBlocks)
      needsFollowUp = true
      
      // 🔑 关键：模型还在流式输出，工具已经开始执行
      if (streamingToolExecutor) {
        for (const toolBlock of msgToolUseBlocks) {
          streamingToolExecutor.addTool(toolBlock, message)
        }
      }
    }
  }
  
  // 即时收割已完成的工具结果（不等流结束）
  if (streamingToolExecutor) {
    for (const result of streamingToolExecutor.getCompletedResults()) {
      if (result.message) {
        yield result.message
        toolResults.push(/* normalized */)
      }
    }
  }
}
```

### 1.4 流式工具并行

`StreamingToolExecutor` 是 Agent Loop 性能优化的核心。一般 Agent 的实现是：等模型说完 → 看有没有工具调用 → 执行 → 结果返回。Claude Code 不等。

```typescript
// src/services/tools/StreamingToolExecutor.ts
export class StreamingToolExecutor {
  private tools: TrackedTool[] = []         // 所有工具的执行追踪
  private hasErrored = false                // Bash 错误时级联取消
  private siblingAbortController: AbortController  // 子控制器（不影响主循环）
  private discarded = false                 // 流式回退时丢弃所有结果
  
  // 模型流式吐出一个 tool_use block，立刻加入执行队列
  addTool(block: ToolUseBlock, message: AssistantMessage): void {
    const isConcurrencySafe = tool.isConcurrencySafe(parsedInput)
    this.tools.push({
      id: block.id,
      block,
      assistantMessage: message,
      status: 'queued',
      isConcurrencySafe,
    })
    void this.processQueue()  // 立即尝试执行
  }
```

**并发控制规则**：

```typescript
private canExecuteTool(isConcurrencySafe: boolean): boolean {
  const executingTools = this.tools.filter(t => t.status === 'executing')
  return (
    executingTools.length === 0 ||
    (isConcurrencySafe && executingTools.every(t => t.isConcurrencySafe))
  )
}
```

翻译成人话：
- **只读工具**（`isConcurrencySafe = true`）：可以和其他只读工具并行
- **写操作工具**（`isConcurrencySafe = false`）：必须独占执行
- 结果按**接收顺序**缓冲 yield，不会乱序

**错误级联**：当 Bash 命令出错时，它会通过 `siblingAbortController` 取消**所有并行的兄弟工具**（因为 Bash 命令之间通常有隐式依赖링），但**不会**取消主循环（不影响 `toolUseContext.abortController`）。其他工具（Read、WebFetch 等）出错则不级联，因为它们是独立的。

```typescript
if (isErrorResult && tool.block.name === BASH_TOOL_NAME) {
  this.hasErrored = true
  this.erroredToolDescription = this.getToolDescription(tool)
  this.siblingAbortController.abort('sibling_error')  // 杀兄弟，不杀主循环
}
```

### 1.5 异常恢复：7 种 continue 路径

`queryLoop` 的 `while(true)` 不只是简单的"有工具调用就继续"。它有 **7 种不同的 continue 原因**，每一种都是一个恢复/续写路径：

| transition.reason | 触发条件 | 行为 |
|---|---|---|
| `'next_turn'` | 正常工具调用完成，需要下一轮 | 消息追加工具结果，turnCount++ |
| `'collapse_drain_retry'` | API 返回 413，context_collapse 有待提交的摘要 | 提交所有暂存的 collapse，重试 API |
| `'reactive_compact_retry'` | 413 或 media 过大，reactive compact 压缩成功 | 用压缩后的消息重试 |
| `'max_output_tokens_escalate'` | 输出 token 限制命中（默认 8K） | 升级到 64K 限制重试（同一请求） |
| `'max_output_tokens_recovery'` | 64K 也不够（最多 3 次） | 注入 "Resume directly — no recap" 消息 |
| `'stop_hook_blocking'` | stop hook 返回阻塞错误 | 把 hook 错误注入消息，重新查询 |
| `'token_budget_continuation'` | token budget 未用完 | 注入 nudge 消息继续生成 |

关键的防死循环设计：
- `hasAttemptedReactiveCompact` 标记：reactive compact 只尝试一次
- `maxOutputTokensRecoveryCount` 计数：最多恢复 3 次
- `transition.reason` 检查：collapse drain 后重试仍失败不再 drain
- autocompact 的 `consecutiveFailures` 熔断器：连续失败 3 次停止尝试

### 1.6 Stop Hooks

当模型输出完成且没有工具调用时（`needsFollowUp = false`），会执行 **stop hooks**，这是 Turn 结束后的生命周期管理：

```typescript
// src/query/stopHooks.ts
export async function* handleStopHooks(...) {
  // 1. Template Job Classification — 分类当前任务类型（await 阻塞等待）
  // 2. Prompt Suggestion — 异步后台生成下一步建议（fire-and-forget）
  // 3. Memory Extraction — 子 Agent 分析对话提取知识（fire-and-forget）
  // 4. Auto-Dream — 后台任务生成（fire-and-forget）
  // 5. Chicago MCP — Computer Use 清理 + 锁释放（await）
  // 6. 用户注册的 executeStopHooks()
}
```

如果 stop hook 返回 `blockingErrors`（例如验证器发现代码错误），Agent Loop 会把这些错误注入消息并 `continue`，让模型修复。

### 1.7 Token Budget 续写

当 `feature('TOKEN_BUDGET')` 开启时，Agent Loop 有一个自动续写机制：

```typescript
if (feature('TOKEN_BUDGET')) {
  const decision = checkTokenBudget(
    budgetTracker!, toolUseContext.agentId,
    getCurrentTurnTokenBudget(), getTurnOutputTokens()
  )
  if (decision.action === 'continue') {
    incrementBudgetContinuationCount()
    state = {
      ...state,
      messages: [...messagesForQuery, ...assistantMessages,
        createUserMessage({ content: decision.nudgeMessage, isMeta: true })],
      transition: { reason: 'token_budget_continuation' },
    }
    continue  // 注入 nudge 消息，继续生成
  }
}
```

当模型在 budget 未用完时就停下了，Agent 会注入一个 meta 消息促使模型继续输出。当检测到 "diminishing returns"（边际递减）时则提前停止。

---

## 2. 上下文管理

### 2.1 总体策略

Claude Code 的上下文管理**不是一刀切**，而是 **5 层机制按顺序依次执行**，前面能搞定就不触发后面：

```
请求进入 queryLoop
   │
   ▼
① Tool Result Budget  ← 聚合级预算裁切（持久化到磁盘）
   │
   ▼
② HISTORY_SNIP       ← 删除噪声消息（直接删，不摘要）
   │
   ▼  
③ Microcompact       ← 缓存层 token 删除（不改消息内容）
   │
   ▼
④ CONTEXT_COLLAPSE   ← 结构化归档（保留结构的 git-log 式摘要）
   │
   ▼
⑤ Autocompact        ← 全量摘要（最后兜底，调模型压缩整个历史）
   │
   ▼
发给 Claude API
```

还有一个**被动触发**的：

```
⑥ Reactive Compact   ← 等 API 返回 413 再压缩（对照 autocompact 的实验）
```

### 2.2 Tool Result Budget

**位置**：`src/utils/toolResultStorage.ts`

在所有压缩之前，首先执行**聚合级别的裁切**：对累计的工具结果施加字节预算，超出的部分持久化到磁盘文件，消息里只保留一个文件路径引用。

```typescript
messagesForQuery = await applyToolResultBudget(
  messagesForQuery,
  toolUseContext.contentReplacementState,
  persistReplacements ? records => void recordContentReplacement(records, agentId) : undefined,
  // 放过那些 maxResultSizeChars = Infinity 的工具（如 FileRead，因为持久化会导致循环读取）
  new Set(tools.filter(t => !Number.isFinite(t.maxResultSizeChars)).map(t => t.name)),
)
```

每个工具都有 `maxResultSizeChars` 属性：
- 大多数工具：有限值（超出存磁盘）
- `FileRead`：`Infinity`（不持久化，避免 Read→file→Read 循环）

### 2.3 HISTORY_SNIP

**Feature-gated**: `feature('HISTORY_SNIP')` — 编译时条件，外部 build 中被移除。

```typescript
if (feature('HISTORY_SNIP')) {
  const snipResult = snipModule!.snipCompactIfNeeded(messagesForQuery)
  messagesForQuery = snipResult.messages
  snipTokensFreed = snipResult.tokensFreed
  if (snipResult.boundaryMessage) yield snipResult.boundaryMessage
}
```

SNIP 是最精细的一层：**直接把某些消息删掉，不做任何摘要**。典型场景：一个工具返回 500 行搜索结果，模型只用了其中 3 行。剩下 497 行就是纯噪声，留着浪费 token，摘要它也是浪费 token，直接删最划算。

`snipTokensFreed` 会传递给后续的 autocompact 阈值计算，以避免在 snip 已经释放了足够空间的情况下误触发 autocompact。

### 2.4 Microcompact

**位置**：`src/services/compact/microCompact.ts`

Microcompact 有**两条路径**，根据场景自动选择：

#### 路径 A：Cached Microcompact（主路径，热缓存时）

**机制**：利用 Claude API 的 `cache_edits` 能力，在**不改消息内容**的情况下告诉 API "这些 token 你缓存里有但别用了"。

```typescript
async function cachedMicrocompactPath(messages, querySource) {
  // 1. 收集所有"可压缩"的工具 ID
  //    可压缩工具：FileRead, Shell, Grep, Glob, WebSearch, WebFetch, FileEdit, FileWrite
  const compactableToolIds = collectCompactableToolIds(messages)
  
  // 2. 按 user message 分组注册
  for (const message of messages) {
    if (message.type === 'user') {
      for (const block of message.content) {
        if (block.type === 'tool_result' && compactableToolIds.has(block.tool_use_id)) {
          mod.registerToolResult(state, block.tool_use_id)
        }
      }
    }
  }
  
  // 3. 决定哪些要删除（基于配置阈值）
  const toolsToDelete = mod.getToolResultsToDelete(state)
  
  // 4. 构造 cache_edits block — 不改消息！
  const cacheEdits = mod.createCacheEditsBlock(state, toolsToDelete)
  
  return { messages /* 原封不动 */, compactionInfo: { pendingCacheEdits: cacheEdits } }
}
```

**关键**：消息内容完全不变（保护 prompt cache），只在 API 请求中附带 `cache_edits` 指令。API 响应会带回 `cache_deleted_input_tokens` 报告实际删除了多少 token。

query.ts 在 API 调用结束后消费这个值：

```typescript
if (feature('CACHED_MICROCOMPACT') && pendingCacheEdits) {
  const usage = lastAssistant?.message.usage
  const cumulativeDeleted = (usage as any).cache_deleted_input_tokens ?? 0
  const deletedTokens = Math.max(0, cumulativeDeleted - pendingCacheEdits.baselineCacheDeletedTokens)
  if (deletedTokens > 0) {
    yield createMicrocompactBoundaryMessage(
      pendingCacheEdits.trigger, 0, deletedTokens, pendingCacheEdits.deletedToolIds, []
    )
  }
}
```

#### 路径 B：Time-Based Microcompact（冷缓存时）

**触发**：距上一次 assistant 消息超过 60 分钟。

```typescript
export function evaluateTimeBasedTrigger(messages, querySource) {
  const config = getTimeBasedMCConfig()
  const lastAssistant = messages.findLast(m => m.type === 'assistant')
  const gapMinutes = (Date.now() - new Date(lastAssistant.timestamp)) / 60_000
  if (gapMinutes >= config.gapThresholdMinutes) return { gapMinutes, config }
}
```

**行为**：保留最近 N 个（默认 5）可压缩的工具结果，其余直接**清空内容**为 `'[Old tool result content cleared]'`。这条路径是**真正修改消息内容**的。

### 2.5 CONTEXT_COLLAPSE

**Feature-gated**: `feature('CONTEXT_COLLAPSE')`

```typescript
if (feature('CONTEXT_COLLAPSE') && contextCollapse) {
  const collapseResult = await contextCollapse.applyCollapsesIfNeeded(
    messagesForQuery, toolUseContext, querySource
  )
  messagesForQuery = collapseResult.messages
}
```

与 autocompact 的区别：**保留结构**。像 git log 一样，每一轮做了什么事、结论是什么都保留为独立的摘要记录，不是混成一坨。

CONTEXT_COLLAPSE 在 autocompact 之前执行。如果 collapse 已经把 token 数降到了阈值以下，autocompact 就是 no-op，保留了更精细的上下文。

**恢复功能**：当 API 返回 413（prompt-too-long）时，CONTEXT_COLLAPSE 有 `recoverFromOverflow()` 方法，会提交所有暂存的 collapse。如果这还不够，则回退到 reactive compact。

### 2.6 Autocompact

**位置**：`src/services/compact/autoCompact.ts` + `src/services/compact/compact.ts` + `src/services/compact/sessionMemoryCompact.ts`

#### 触发条件

```typescript
// 阈值计算
const AUTOCOMPACT_BUFFER_TOKENS = 13_000
function getAutoCompactThreshold(model: string): number {
  return getEffectiveContextWindowSize(model) - AUTOCOMPACT_BUFFER_TOKENS
}

// 判断是否触发
async function shouldAutoCompact(messages, model, querySource, snipTokensFreed) {
  if (querySource === 'session_memory' || querySource === 'compact') return false  // 防递归
  if (feature('CONTEXT_COLLAPSE') && querySource === 'marble_origami') return false  // 防 ctx-agent 破坏主线程 collapse 状态
  if (!isAutoCompactEnabled()) return false
  // Reactive-only 模式：抑制主动 autocompact，让 API 413 触发 reactive compact
  if (feature('REACTIVE_COMPACT') && getFeatureValue('tengu_cobalt_raccoon', false)) return false
  // Context-collapse 模式：collapse 接管上下文管理，抑制 autocompact 防止竞态
  if (feature('CONTEXT_COLLAPSE') && isContextCollapseEnabled()) return false
  
  const tokenCount = tokenCountWithEstimation(messages) - snipTokensFreed
  return tokenCount >= getAutoCompactThreshold(model)
}
```

**熔断器**：连续失败 3 次后停止尝试：

```typescript
if (consecutiveFailures >= MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES) {
  logForDebugging('Autocompact circuit breaker tripped')
  return { wasCompacted: false }
}
```

#### 两条压缩路径：Session Memory Compact vs Full Compact

当 autocompact 被触发时（`autoCompactIfNeeded()`），并**不是**直接调用 `compactConversation()`。而是先尝试 **Session Memory Compact**，只有在它不可用时才回退到 **Full Compact**（`compactConversation()`）：

```typescript
// autoCompact.ts — autoCompactIfNeeded() 的核心逻辑
// EXPERIMENT: Try session memory compaction first
const sessionMemoryResult = await trySessionMemoryCompaction(
  messages, toolUseContext.agentId, recompactionInfo.autoCompactThreshold,
)
if (sessionMemoryResult) {
  setLastSummarizedMessageId(undefined)
  runPostCompactCleanup(querySource)
  markPostCompaction()
  return { wasCompacted: true, compactionResult: sessionMemoryResult }
}

// Session memory 不可用，回退到 Full Compact
const compactionResult = await compactConversation(
  messages, toolUseContext, cacheSafeParams,
  true, undefined, true, recompactionInfo,
)
```

手动 `/compact` 命令也是同样的优先级：先尝试 SM-compact，失败后 fallback 到 full compact（但带自定义指令时直接走 full compact，因为 SM-compact 不支持自定义指令）。

---

#### 路径 A：Session Memory Compact（SM-Compact）

**位置**：`src/services/compact/sessionMemoryCompact.ts` + `src/services/SessionMemory/`

SM-Compact 是一种**压缩时刻免模型调用**的压缩方式：它不需要在压缩触发时 fork 子 Agent 去调用 Claude API 做摘要，而是直接读取一个**已经在后台维护好的** Session Memory 文件作为摘要。这使得**压缩步骤本身**速度极快（毫秒级）、无需额外 API 调用。

> **成本澄清**：SM-Compact 的"零成本"仅指压缩那一刻不调 API。生成和维护 Session Memory 文件的工作由后台 sub-agent 完成，**这个后台提取过程本身是有 API 成本的**（通过 `runForkedAgent()` 调用 Claude）。SM-Compact 的本质是把"生成摘要"的成本从压缩时刻**前移**到日常后台维护中，压缩时只是取用已有成果。

##### Session Memory 是什么

Session Memory 是一个**独立于 Compact、持续在后台运行的对话信息提取系统**。它注册为 `postSamplingHook`，在每次 Agent turn 完成后（`handleStopHooks` 阶段）检查触发条件。满足条件时，它 fork 一个 sub-agent **调用 Claude API** 从最近的对话中提取关键信息，用 `FileEditTool` 增量更新一个结构化的 Markdown 笔记文件：

```
~/.claude/projects/<hash>/session-memory/session-notes.md
```

文件遵循固定模板结构（10 个章节），由后台 sub-agent 用 `FileEditTool` 增量更新：

```markdown
# Session Title
_A short and distinctive 5-10 word descriptive title for the session_

# Current State
_What is actively being worked on right now?_

# Task specification
_What did the user ask to build?_

# Files and Functions
_What are the important files?_

# Workflow
_What bash commands are usually run?_

# Errors & Corrections
_Errors encountered and how they were fixed_

# Codebase and System Documentation
_Important system components_

# Learnings
_What has worked well? What has not?_

# Key results
_Exact results if the user asked for specific output_

# Worklog
_Step by step, what was attempted and done?_
```

##### Session Memory 提取触发条件

```typescript
// sessionMemory.ts — shouldExtractMemory()
export function shouldExtractMemory(messages: Message[]): boolean {
  const currentTokenCount = tokenCountWithEstimation(messages)
  
  // 第一次：达到初始化阈值（默认 10K tokens）才开始
  if (!isSessionMemoryInitialized()) {
    if (!hasMetInitializationThreshold(currentTokenCount)) return false
    markSessionMemoryInitialized()
  }
  
  // 后续：需要同时满足 token 增长阈值 AND 工具调用数阈值
  const hasMetTokenThreshold = hasMetUpdateThreshold(currentTokenCount)  // 默认 5K tokens 增长
  const toolCallsSinceLastUpdate = countToolCallsSince(messages, lastMemoryMessageUuid)
  const hasMetToolCallThreshold = toolCallsSinceLastUpdate >= getToolCallsBetweenUpdates()  // 默认 3 次
  
  // 两种触发方式：
  //   1. token + tool call 双阈值都满足
  //   2. token 阈值满足 + 最后一轮没有工具调用（自然对话断点）
  return (hasMetTokenThreshold && hasMetToolCallThreshold) ||
         (hasMetTokenThreshold && !hasToolCallsInLastTurn)
}
```

提取 Agent 用 `runForkedAgent()` 运行（共享 prompt cache，降低 API 成本），只允许使用 `FileReadTool` 和 `FileEditTool`（限于 session-memory 目录）。提取 Agent 的 prompt 明确要求它**只更新有变化的章节**，保留模板结构不动，并使用并行 Edit 调用以最少 turn 完成更新。每个章节内容有 ~2000 token 上限，总文件不超过 12000 token。

**关键**：Session Memory 提取和 SM-Compact 是**解耦的两个系统**。Session Memory 在后台持续运行、持续消耗 API 调用来维护笔记文件。SM-Compact 只在 autocompact 触发时才被调用，它只是"消费"已有的 Session Memory 文件。即使不使用 SM-Compact（比如 feature gate 关闭），Session Memory 提取仍然会运行（由独立的 `tengu_session_memory` gate 控制）。

##### 从 Session Memory 到 SM-Compact：压缩时发生了什么

当 autocompact 触发、SM-Compact 被选中时，发生的事情是：

1. **读取已有的 Session Memory 文件**（`getSessionMemoryContent()`），不调用 API
2. **确定分割点**：通过 `lastSummarizedMessageId` 知道 Session Memory 覆盖到了哪条消息
3. **计算保留窗口**（`calculateMessagesToKeepIndex()`）：从 `lastSummarizedMessageId + 1` 开始计算未覆盖的消息够不够（≥10K tokens 且 ≥5 条文本消息）。如果不够，**会向前扩展到已覆盖区间**，纳入更早的消息直到满足最低要求或碰到 40K 上限
4. **丢弃更早的头部**：保留窗口之前的原始对话消息被丢弃，用 Session Memory 文件内容作为摘要替代
5. 组装：`[CompactBoundary] + [Session Memory 作为摘要] + [保留窗口的原始消息] + [附件]`

本质上就是：**用后台 sub-agent 提前维护好的笔记替换掉头部的原始对话历史，同时保留一个足够大的尾部窗口确保上下文连续性**。注意这个尾部窗口可能包含一些已经被 Session Memory 覆盖的消息（如果未覆盖部分太短），此时这些消息的信息会在上下文中同时存在两份（摘要中一份 + 原始消息一份）。

```
时间线示意：

  Turn 1  Turn 2  Turn 3  Turn 4  Turn 5  Turn 6  Turn 7
  ─────── ─────── ─────── ─────── ─────── ─────── ───────
                  ↑                       ↑
                  Session Memory          Session Memory
                  第一次提取               第二次提取
                  (调 API, 有成本)          (调 API, 有成本)
                  覆盖 Turn 1-3            覆盖 Turn 1-6
                  ↓                       ↓
                  session-notes.md v1      session-notes.md v2
                                          lastSummarizedMessageId → Turn 6 末尾

  假设 Turn 7 触发 autocompact:

  情况 A: Turn 7 内容足够多（≥10K tokens 且 ≥5 条文本消息）
  ┌────────────────────────────────────────────────────────┐
  │ SM-Compact 做的事：                                     │
  │  丢弃: Turn 1-6 的原始消息（已被 session-notes.md 覆盖）  │
  │  保留: session-notes.md v2 + Turn 7 消息                 │
  │  结果: [Boundary] + [session-notes.md] + [Turn 7 消息]   │
  └────────────────────────────────────────────────────────┘

  情况 B: Turn 7 内容太少（<10K tokens），需要向前扩展
  ┌────────────────────────────────────────────────────────┐
  │ SM-Compact 做的事：                                     │
  │  丢弃: Turn 1-4 的原始消息                               │
  │  保留: session-notes.md v2 + Turn 5-7 消息               │
  │  注意: Turn 5-6 已被 Session Memory 覆盖，但因为 Turn 7   │
  │        不够 10K tokens，所以向前扩展保留了它们的原始消息。  │
  │        此时 Turn 5-6 的信息在上下文中存在两份：            │
  │        一份在 session-notes.md 摘要中，一份是原始消息。    │
  │  结果: [Boundary] + [session-notes.md] + [Turn 5-7 消息] │
  └────────────────────────────────────────────────────────┘

  两种情况下 API 调用都是 0。

  如果 SM-Compact 不可用，Full Compact 做的事：
  ┌────────────────────────────────────────────────────────┐
  │  把 Turn 1-7 全部喂给一个 fork 子 Agent                  │
  │  子 Agent 调 Claude API 生成 9 章节摘要                   │
  │  丢弃所有原始消息，只留摘要（不保留任何原始消息）           │
  │  结果: [Boundary] + [模型生成的摘要]                      │
  │  API 调用: 1（摘要生成）                                  │
  └────────────────────────────────────────────────────────┘
```

##### SM-Compact 的详细逻辑

```typescript
// sessionMemoryCompact.ts — trySessionMemoryCompaction()
export async function trySessionMemoryCompaction(messages, agentId?, autoCompactThreshold?) {
  // 1. Feature gate 检查
  if (!shouldUseSessionMemoryCompaction()) return null  // 需要 tengu_session_memory AND tengu_sm_compact

  // 2. 等待正在进行的 Session Memory 提取完成（最多等 15 秒，超过 1 分钟的视为僵尸）
  await waitForSessionMemoryExtraction()

  // 3. 读取 Session Memory 内容
  const sessionMemory = await getSessionMemoryContent()
  if (!sessionMemory) return null                      // 文件不存在
  if (await isSessionMemoryEmpty(sessionMemory)) return null  // 只有模板没有实际内容

  // 4. 确定分割点：哪些消息已被 Session Memory 覆盖
  let lastSummarizedIndex = messages.findIndex(msg => msg.uuid === lastSummarizedMessageId)
  // 如果是恢复的会话（没有 lastSummarizedMessageId），设为最后一条消息

  // 5. 计算保留哪些消息
  const startIndex = calculateMessagesToKeepIndex(messages, lastSummarizedIndex)
  const messagesToKeep = messages.slice(startIndex).filter(m => !isCompactBoundaryMessage(m))

  // 6. 用 Session Memory 内容直接构建压缩结果（不调用 API！）
  return createCompactionResultFromSessionMemory(messages, sessionMemory, messagesToKeep, ...)
}
```

##### 消息保留策略（calculateMessagesToKeepIndex）

SM-Compact 不是把所有历史消息都丢弃，而是保留一个尾部窗口。保留策略有三个维度：

```typescript
const DEFAULT_SM_COMPACT_CONFIG = {
  minTokens: 10_000,           // 至少保留 10K tokens 的消息
  minTextBlockMessages: 5,      // 至少保留 5 条含文本的消息
  maxTokens: 40_000,           // 最多保留 40K tokens（硬上限）
}
```

算法逻辑：
1. 先计算 `lastSummarizedIndex + 1` 到末尾的消息（这些是 Session Memory **尚未覆盖**的新消息）
2. 如果这个区间已经满足 `minTokens`（10K）**且** `minTextBlockMessages`（5 条）→ 够了，只保留这些未覆盖的消息
3. 如果**不够**（比如未覆盖区间只有 2K tokens 和 2 条消息）→ **向前扩展到已被 Session Memory 覆盖的区间**，纳入更早的消息，直到满足两个最低要求
4. 但不超过 `maxTokens`（40K）硬上限
5. 不越过上一次 compact boundary（磁盘不连续点）
6. 最后调整起始索引，确保不拆分 `tool_use` / `tool_result` 配对，也不拆分共享 `message.id` 的流式消息（thinking block + tool_use block 可能有相同的 `message.id` 但不同的 `uuid`）

**关键**：`lastSummarizedIndex` **不是硬边界**。当未覆盖的尾部不够大时，算法会把一些**已经被 Session Memory 覆盖的消息**也保留为原始形式。这意味着压缩后的上下文中，这些消息与 Session Memory 的摘要**同时存在**，形成局部冗余，但保证了上下文的连续性和完整性。

##### SM-Compact 的安全阀

```typescript
// 如果压缩后的消息仍然超过 autocompact 阈值 → 放弃，回退到 Full Compact
if (autoCompactThreshold !== undefined && postCompactTokenCount >= autoCompactThreshold) {
  logEvent('tengu_sm_compact_threshold_exceeded', { postCompactTokenCount, autoCompactThreshold })
  return null
}
```

##### SM-Compact vs Full Compact 对比

| 维度 | Session Memory Compact | Full Compact |
|---|---|---|
| **压缩时 API 调用** | 无（摘要已由后台 Session Memory 提取预备好） | 需要 fork 子 Agent 调 Claude API 实时生成 |
| **总系统成本** | Session Memory 后台提取有 API 成本（前置支付） | 仅在压缩时产生成本 |
| **压缩耗时** | 毫秒级（读文件 + 切消息） | 5-10+ 秒（等待 API 响应） |
| **摘要来源** | 已有的 Session Memory 文件（后台 sub-agent 增量维护） | 实时调用模型对话历史做全量摘要 |
| **消息保留** | 保留尾部窗口（10K-40K tokens） | 全部丢弃，只留摘要 |
| **摘要结构** | 固定 10 章节 Markdown 模板 | 模型自由生成的 9 章节摘要 |
| **文件恢复** | 仅注入 Plan（如有） | 恢复最多 5 个最近文件 + Deferred tools + Agent listing + MCP + Skill |
| **前提条件** | Session Memory 已提取且非空 | 无（始终可用） |
| **自定义指令** | 不支持 | 支持（`/compact <instructions>`） |

---

#### 路径 B：Full Compact（compactConversation）

当 SM-Compact 不可用时（feature gate 关闭、Session Memory 为空、压缩后仍超阈值等），回退到传统的 Full Compact。这是一个**需要调用 Claude API 的完整压缩流水线**：

```
1. PRE-COMPACT HOOKS
   └─ 执行用户注册的 preCompact hook（可注入额外指令）

2. STREAM SUMMARY（带 PTL 重试）
   └─ fork 一个子 Agent，调用 Claude API 对历史做摘要
   └─ 如果摘要请求本身也 413 → 截断头部重试（最多 3 次）

3. CLEAR CACHES
   └─ readFileState.clear()（文件读取缓存全部失效）
   └─ loadedNestedMemoryPaths.clear()
   
4. RESTORE FILES（最多 5 个，每个 5K tokens，总 50K）
   └─ 摘要后重新注入最近编辑/读取的文件内容

5. POST-COMPACT ATTACHMENTS
   ├─ Deferred tools 重新声明
   ├─ Agent listing
   ├─ MCP instructions
   ├─ Plan file + Plan mode instructions
   └─ Skill attachments

6. SESSION START HOOKS
   └─ 以 'compact' 触发器执行

7. BOUNDARY + SUMMARY MESSAGES
   ├─ CompactBoundaryMessage（标记压缩点）
   └─ UserMessage（isCompactSummary: true）

8. POST-COMPACT HOOKS
   └─ 执行用户注册的 postCompact hook

返回：{ boundaryMarker, summaryMessages, attachments, hookResults,
        preCompactTokenCount, postCompactTokenCount, compactionUsage }
```

##### streamCompactSummary 的双路径

Full Compact 的核心是 `streamCompactSummary()`，它有两条执行路径：

```typescript
// compact.ts — streamCompactSummary()
// 路径 1（默认）：forked agent — 复用主对话的 prompt cache
const result = await runForkedAgent({
  promptMessages: [summaryRequest],
  cacheSafeParams,           // 共享 system prompt + 消息前缀 → cache key 一致
  canUseTool: createCompactCanUseTool(),
  querySource: 'compact',
  maxTurns: 1,
  skipCacheWrite: true,
  overrides: { abortController: context.abortController },
})

// 路径 2（fallback）：直接流式调用 — cache miss 但保证可用
const streamingGen = queryModelWithStreaming({
  messages: normalizeMessagesForAPI([...messages, summaryRequest], tools),
  tools: useToolSearch ? [FileReadTool, ToolSearchTool, ...mcpTools] : [FileReadTool],
  maxOutputTokensOverride: COMPACT_MAX_OUTPUT_TOKENS,
  // ...
})
```

**Prompt 结构**：摘要请求使用一个精心设计的 prompt（约 3K tokens），要求模型先在 `<analysis>` 标签中分析（作为草稿，最终被 `formatCompactSummary()` 剥离），然后在 `<summary>` 标签中输出 9 章节的结构化摘要：
1. Primary Request and Intent
2. Key Technical Concepts
3. Files and Code Sections（含完整代码片段）
4. Errors and fixes
5. Problem Solving
6. All user messages（逐条列出非工具结果的用户消息）
7. Pending Tasks
8. Current Work（最近工作的精确描述）
9. Optional Next Step

**PTL 重试**：如果摘要请求本身也触发 prompt-too-long（对话太长连摘要都装不下），会截断头部消息重试，最多 3 次。

##### Partial Compact（partialCompactConversation）

除了全量压缩，还有**部分压缩**，支持两个方向：

```typescript
// compact.ts — partialCompactConversation()
// direction = 'from': 压缩选定消息之后的部分，保留之前的（prompt cache 友好）
// direction = 'up_to': 压缩选定消息之前的部分，保留之后的（cache 失效）
```

这用于用户在 UI 中选择某条消息后执行部分压缩，保留更精细的控制。

#### 消息组装顺序

两条路径最终都调用 `buildPostCompactMessages()` 组装结果：

```typescript
// src/services/compact/compact.ts
export function buildPostCompactMessages(result: CompactionResult): Message[] {
  return [
    result.boundaryMarker,      // 压缩边界标记
    ...result.summaryMessages,   // 摘要内容（SM: Session Memory / Full: 模型生成的摘要）
    ...(result.messagesToKeep ?? []),  // SM-Compact 保留的尾部消息 / Full Compact 为空
    ...result.attachments,       // 恢复的文件内容 + hook 消息
    ...result.hookResults,       // session start hook 输出
  ]
}
```

注意 `messagesToKeep`：SM-Compact 保留了尾部窗口的原始消息，Full Compact 则没有（全部被摘要替代）。

### 2.7 Reactive Compact

**Feature-gated**: `feature('REACTIVE_COMPACT')`

与 autocompact 的根本区别：autocompact 是**主动预防**（在 token 快满时提前压缩），reactive compact 是**被动响应**（等 API 真的返回 413 错误再压缩）。

当 reactive compact 启用时，它会**抑制** autocompact 的主动触发，让 API 错误自然发生，然后在 query.ts 的错误恢复路径中处理：

```typescript
// query.ts — 模型返回后的恢复逻辑
if ((isWithheld413 || isWithheldMedia) && reactiveCompact) {
  const compacted = await reactiveCompact.tryReactiveCompact({
    hasAttempted: hasAttemptedReactiveCompact,
    querySource,
    aborted: toolUseContext.abortController.signal.aborted,
    messages: messagesForQuery,
    cacheSafeParams: { systemPrompt, userContext, systemContext, toolUseContext, 
                       forkContextMessages: messagesForQuery },
  })
  
  if (compacted) {
    const postCompactMessages = buildPostCompactMessages(compacted)
    for (const msg of postCompactMessages) yield msg
    state = { ...state, messages: postCompactMessages, hasAttemptedReactiveCompact: true }
    continue  // 重试 API 调用
  }
  
  // 恢复失败 — 原样暴露错误，终止
  yield lastMessage
  return { reason: 'prompt_too_long' }
}
```

**关键安全措施**：暂扣 413 错误（streaming 循环中检测到但不 yield），直到恢复逻辑决定是否能处理。如果 withhold 了但没 recover（比如 reactive compact 也失败了），最后才 yield 错误消息。

### 2.8 压缩流水线的完整代码路径

以下是每次 query loop 迭代的压缩决策**完整路径**：

```
messagesForQuery (原始消息)
     │
     ├─① applyToolResultBudget()
     │   └─ 超限工具结果 → 持久化到磁盘，保留路径引用
     │
     ├─② snipCompactIfNeeded()                    [HISTORY_SNIP]
     │   └─ 噪声消息 → 直接删除
     │   └─ snipTokensFreed → 传给 autocompact
     │
     ├─③ microcompact()
     │   ├─ 热缓存 → cachedMicrocompactPath()
     │   │   └─ 构造 cache_edits → 不改消息
     │   └─ 冷缓存 (>60min gap) → timeBasedMicrocompactPath()
     │       └─ 清空旧工具结果内容
     │
     ├─④ applyCollapsesIfNeeded()                  [CONTEXT_COLLAPSE]
     │   └─ 旧轮次 → 结构化摘要（保留轮次结构）
     │
     ├─⑤ autoCompactIfNeeded()
     │   ├─ shouldAutoCompact() = false → 跳过
     │   ├─ shouldAutoCompact() = true →
     │   │   ├─ 路径 A: trySessionMemoryCompaction()（SM-Compact，零 API 调用）
     │   │   │   ├─ gate 检查 (tengu_session_memory + tengu_sm_compact)
     │   │   │   ├─ waitForSessionMemoryExtraction()（等待后台提取完成，最多 15s）
     │   │   │   ├─ 读取 Session Memory 文件内容作为摘要
     │   │   │   ├─ calculateMessagesToKeepIndex()（保留尾部 10K-40K tokens）
     │   │   │   ├─ 安全阀：压缩后仍超阈值 → return null → 回退路径 B
     │   │   │   └─ 成功 → buildPostCompactMessages() → 使用
     │   │   │
     │   │   └─ 路径 B: compactConversation()（Full Compact，调用 Claude API）
     │   │       ├─ preCompactHooks
     │   │       ├─ streamCompactSummary (fork 子 Agent，共享 prompt cache)
     │   │       │   └─ PTL 重试（摘要请求本身 413 → 截断头部，最多 3 次）
     │   │       ├─ readFileState.clear() + loadedNestedMemoryPaths.clear()
     │   │       ├─ createPostCompactFileAttachments（恢复最近 5 个文件）
     │   │       ├─ Deferred tools / Agent listing / MCP / Plan / Skill 重新声明
     │   │       ├─ processSessionStartHooks('compact')
     │   │       └─ postCompactHooks
     │   └─ 熔断（3 次连续失败）→ 跳过
     │
     ├─⑥ [阻塞检查]
     │   └─ 如果 autocompact 关闭且 token 到了阻塞限制
     │       → yield 错误消息 → return
     │
     └─→ messagesForQuery (处理后) → 发给 Claude API

API 返回后的响应式路径：
     │
     ├─ 413 (prompt-too-long)
     │   ├─ ④ contextCollapse.recoverFromOverflow()  → continue
     │   └─ ⑥ reactiveCompact.tryReactiveCompact()   → continue 或 return
     │
     ├─ max_output_tokens
     │   ├─ 升级到 64K → continue
     │   └─ 注入恢复消息 (max 3x) → continue 或 yield 错误
     │
     └─ media_size_error
         └─ reactiveCompact strip-retry → continue 或 return
```

---

## 3. 工具系统

### 3.1 Tool 接口：一切皆 buildTool()

Claude Code 完全没有继承。40+ 工具全是纯函数式的 `buildTool()` 工厂函数：

```typescript
// src/Tool.ts — 工厂函数
export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return {
    ...TOOL_DEFAULTS,
    userFacingName: () => def.name,
    ...def,
  } as BuiltTool<D>
}

// 默认值
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: (_input?: unknown) => false,  // 默认不允许并发
  isReadOnly: (_input?: unknown) => false,
  isDestructive: (_input?: unknown) => false,
  checkPermissions: (input, _ctx?) => Promise.resolve({ behavior: 'allow', updatedInput: input }),
  toAutoClassifierInput: (_input?: unknown) => '',
}
```

每个工具文件的标准写法：

```typescript
export const MyTool = buildTool({
  name: 'my_tool',
  inputSchema: z.object({
    param1: z.string().describe('参数说明'),
    param2: z.number().optional(),
  }),

  // 动态描述（可根据输入/权限变化）
  async description(input, { isNonInteractiveSession, toolPermissionContext, tools }) {
    return '这个工具做什么...'
  },

  // System prompt 部分（告诉模型何时/如何使用此工具）
  async prompt({ getToolPermissionContext, tools, agents }) {
    return '使用指南...'
  },

  // 输入校验（Zod 内置 + 自定义）
  async validateInput(input, context) {
    return { result: true }
  },

  // 权限检查
  async checkPermissions(input, context) {
    return { behavior: 'allow', updatedInput: input }
  },

  // 核心执行
  async call(args, context, canUseTool, parentMessage, onProgress) {
    // ... 实际工作
    return { data: result }
  },

  // 并发安全性标记
  isConcurrencySafe(input) { return true },  // 只读工具返回 true
  isReadOnly(input) { return true },

  // 搜索/读取折叠（UI 用）
  isSearchOrReadCommand(input) {
    return { isSearch: false, isRead: true }
  },

  // UI 渲染名
  userFacingName(input) { return 'MyTool' },
  
  // 结果序列化（给 API 的 tool_result）
  mapToolResultToToolResultBlockParam(content, toolUseID) {
    return { type: 'tool_result', tool_use_id: toolUseID, content: JSON.stringify(content) }
  },
  
  // 结果 UI 渲染（给终端的 React 组件）
  renderToolResultMessage(content, progressMessages, options) {
    return <MyToolResult data={content} />
  },
  
  // 压缩时的摘要
  getToolUseSummary(input) { return `MyTool(${input.param1})` },
  
  // 活动描述（Spinner 显示）
  getActivityDescription(input) { return `正在处理 ${input.param1}` },
  
  // 权限匹配器（hook if 条件用）
  async preparePermissionMatcher(input) {
    return (pattern: string) => minimatch(input.param1, pattern)
  },
  
  // 自动分类器输入（安全审计用）
  toAutoClassifierInput(input) { return input.param1 },
  
  maxResultSizeChars: 30_000,  // 超出存磁盘
})
```

**完全自包含**：schema、权限、执行逻辑、UI 渲染、压缩摘要、活动描述，全在一个文件里。没有全局状态泄露。

### 3.2 工具注册：三阶段动态组装

**位置**：`src/tools.ts`

工具池**不是静态的**，每个 session 动态组装：

```typescript
// 阶段一：直接导入（始终可用）
import { AgentTool } from './tools/AgentTool/AgentTool.js'
import { BashTool } from './tools/BashTool/BashTool.js'
import { FileEditTool } from './tools/FileEditTool/FileEditTool.js'
import { FileReadTool } from './tools/FileReadTool/FileReadTool.js'
import { FileWriteTool } from './tools/FileWriteTool/FileWriteTool.js'
// ... ~15 个核心工具

// 阶段二：Feature-gated 条件导入
const SleepTool = feature('PROACTIVE') || feature('KAIROS')
  ? require('./tools/SleepTool/SleepTool.js').SleepTool : null

const SnipTool = feature('HISTORY_SNIP')
  ? require('./tools/SnipTool/SnipTool.js').SnipTool : null

const REPLTool = process.env.USER_TYPE === 'ant'
  ? require('./tools/REPLTool/REPLTool.js').REPLTool : null

// 阶段三：getAllBaseTools() 动态组装
export function getAllBaseTools(): Tools {
  return [
    AgentTool,
    TaskOutputTool,
    BashTool,
    ...(hasEmbeddedSearchTools() ? [] : [GlobTool, GrepTool]),
    FileReadTool, FileEditTool, FileWriteTool,
    NotebookEditTool, WebFetchTool,
    ...(SleepTool ? [SleepTool] : []),
    ...(SnipTool ? [SnipTool] : []),
    ...(isTodoV2Enabled() ? [TaskCreateTool, TaskGetTool, ...] : []),
    ...(isToolSearchEnabledOptimistic() ? [ToolSearchTool] : []),
    ListMcpResourcesTool, ReadMcpResourceTool,
    // ... 40+ 工具
  ]
}
```

最终组装：内置工具 + MCP 工具合并，去重，**按名称排序**保证 prompt cache 稳定性：

```typescript
export function assembleToolPool(permissionContext, mcpTools): Tools {
  const builtInTools = getTools(permissionContext)  // 过滤权限后的内置工具
  const allowedMcpTools = filterToolsByDenyRules(mcpTools, permissionContext)
  
  const byName = (a, b) => a.name.localeCompare(b.name)
  return uniqBy(
    [...builtInTools].sort(byName).concat(allowedMcpTools.sort(byName)),
    'name',
  )
}
```

**Deferred Loading**：启用 `ToolSearch` 后，部分工具标记 `shouldDefer: true`，schema 不发送给模型（降低 prompt token），需要时通过 ToolSearch 工具查找后才能使用。`alwaysLoad: true` 标记的工具（如 MCP 的 `_meta['anthropic/alwaysLoad']`）永远不被延迟。

### 3.3 工具执行：从权限到结果的完整链路

**位置**：`src/services/tools/toolExecution.ts`

一个 `tool_use` block 从 Claude 输出到结果回注的完整路径：

```
Claude 输出: { type: 'tool_use', id: 'xxx', name: 'Bash', input: { command: 'ls' } }
    │
    ▼
  runToolUse() — 异步生成器
    │
    ├─ 1. 查找工具定义
    │     └─ findToolByName(tools, name)，支持 alias 回退
    │     └─ 找不到 → yield 错误 tool_result
    │
    ├─ 2. 输入校验（两层）
    │     ├─ Zod schema.safeParse(input) → 类型校验
    │     │   └─ 失败时如果是 deferred 工具 → 提示 "schema not sent, use ToolSearch first"
    │     └─ tool.validateInput(input, context) → 业务校验
    │
    ├─ 3. Pre-Tool-Use Hooks
    │     └─ 可能返回：hookPermissionResult, hookUpdatedInput, preventContinuation
    │
    ├─ 4. 权限决策
    │     └─ resolveHookPermissionDecision()
    │         ├─ hook 已决定？→ 使用 hook 决定
    │         ├─ 否则 → canUseTool() → 用户确认对话框
    │         └─ 拒绝 → yield REJECT_MESSAGE
    │
    ├─ 5. 工具执行（带 OpenTelemetry tracing）
    │     ├─ startToolSpan(tool.name, ...)
    │     ├─ result = await tool.call(input, context, canUseTool, parentMessage, onProgress)
    │     └─ endToolSpan()
    │
    ├─ 6. Post-Tool-Use Hooks
    │     └─ 可能修改工具输出（MCP 工具的 updatedMCPToolOutput）
    │
    └─ 7. 结果序列化 + yield
          ├─ tool.mapToolResultToToolResultBlockParam(output, toolUseID)
          └─ yield { message: createUserMessage({ content: [toolResultBlock] }) }
```

**权限决策来源追踪**（OTel）：

```
解锁来源            | OTel Classification
────────────────────┼─────────────────────
Hook 直接批准        | hook
会话级临时规则 allow  | user_temporary
本地设置规则 allow    | user_permanent
用户设置规则 allow    | user_permanent
用户交互确认         | 使用 tool 返回的 decisionClassification
默认配置             | config
```

### 3.4 并发控制详解

有两套并发执行路径：

#### 路径 A：StreamingToolExecutor（模型流式输出时）

模型还在流式输出后面的内容时，前面的工具就已经在跑了。

```typescript
// 核心并发规则
class StreamingToolExecutor {
  private canExecuteTool(isConcurrencySafe: boolean): boolean {
    const executing = this.tools.filter(t => t.status === 'executing')
    return (
      executing.length === 0 ||  // 没人在跑 → 可以
      (isConcurrencySafe && executing.every(t => t.isConcurrencySafe))  // 都是只读 → 可以
    )
  }
  
  private async processQueue(): Promise<void> {
    for (const tool of this.tools) {
      if (tool.status !== 'queued') continue
      if (this.canExecuteTool(tool.isConcurrencySafe)) {
        await this.executeTool(tool)
      } else if (!tool.isConcurrencySafe) {
        break  // 写操作必须等前面全部完成，停止处理队列
      }
    }
  }
}
```

**结果 yield 顺序保证**：

```typescript
*getCompletedResults(): Generator<MessageUpdate> {
  for (const tool of this.tools) {
    // progress 消息总是立即 yield（不等工具完成）
    while (tool.pendingProgress.length > 0) {
      yield { message: tool.pendingProgress.shift()! }
    }
    if (tool.status === 'yielded') continue
    if (tool.status === 'completed' && tool.results) {
      tool.status = 'yielded'
      for (const message of tool.results) {
        yield { message }
      }
    } else if (tool.status === 'executing' && !tool.isConcurrencySafe) {
      break  // 非并发工具挡住后续所有工具的结果输出
    }
  }
}
```

#### 路径 B：toolOrchestration（模型输出完成后的批量执行）

```typescript
// src/services/tools/toolOrchestration.ts
export async function* runTools(toolUseBlocks, assistantMessages, canUseTool, toolUseContext) {
  // 将工具调用分区为批次
  for (const { isConcurrencySafe, blocks } of partitionToolCalls(toolUseBlocks, toolUseContext)) {
    if (isConcurrencySafe) {
      // 只读批次 → 全部并发执行
      for await (const update of runToolsConcurrently(blocks, ...)) {
        // 上下文修改器缓存（不立即应用，避免竞态）
        if (update.contextModifier) queuedModifiers[toolUseID].push(update.contextModifier)
        yield update
      }
      // 批次完成后才应用上下文修改
      for (const block of blocks) {
        for (const modifier of queuedModifiers[block.id]) {
          currentContext = modifier(currentContext)
        }
      }
    } else {
      // 写操作批次 → 串行执行
      for await (const update of runToolsSerially(blocks, ...)) {
        if (update.newContext) currentContext = update.newContext
        yield update
      }
    }
  }
}
```

**分区策略**：连续的只读工具合并为一个并发批次，遇到写操作就切断。

### 3.5 BashTool：最复杂的单个工具

`src/tools/BashTool/` 目录包含 **18 个文件**，总代码量 2000+ 行。它做的远不止 `exec(command)`。

#### 文件清单

| 文件 | 职责 |
|---|---|
| `BashTool.tsx` | 主工具定义（~1000 行） |
| `bashCommandHelpers.ts` | 命令解析辅助函数 |
| `bashPermissions.ts` | 权限规则匹配 |
| `bashSecurity.ts` | 安全验证 |
| `BashToolResultMessage.tsx` | 结果渲染组件 |
| `commandSemantics.ts` | 退出码语义解释 |
| `commentLabel.ts` | 行内注释提取 |
| `destructiveCommandWarning.ts` | 危险操作警告 |
| `modeValidation.ts` | 沙箱模式检查 |
| `pathValidation.ts` | 路径规范化 |
| `prompt.ts` | 工具 prompt 生成 |
| `readOnlyValidation.ts` | 只读检测 |
| `sedEditParser.ts` | sed 编辑解析 + 预览 |
| `sedValidation.ts` | sed 命令验证 |
| `shouldUseSandbox.ts` | 沙箱决策逻辑 |
| `toolName.ts` | 工具名常量 |
| `UI.tsx` | UI 渲染组件 |
| `utils.ts` | 图片调整、输出处理 |

#### 安全层

**1. 命令解析 + 分类**

```typescript
// bashSecurity.ts
const parsed = await parseForSecurity(command)
if (parsed.kind !== 'simple') {
  // 太复杂/畸形：保守处理，交给 hook
  return () => true
}
// 复合命令 (ls && git push) 逐段判定安全性
const subcommands = parsed.commands.map(c => c.argv.join(' '))
```

**2. 沙箱执行**

```typescript
// shouldUseSandbox.ts
// macOS → sandbox-exec 沙箱
// Linux → seccomp
// 配置：tengu_sandbox_disabled_commands (Statsig)
// 用户覆盖：settings.json > sandbox.excludedCommands（支持通配符）
```

**3. 读写分类**（决定并发安全性）

```typescript
// readOnlyValidation.ts
isReadOnly(input): boolean
// grep, find, cat, head, tail, wc, ls, tree → 只读
// git push, rm, mv, npm install → 非只读
// 含 cd → 非只读（改变工作目录）
```

**4. sed 命令特殊处理**

```typescript
// sedEditParser.ts + sedValidation.ts
// 检测到 sed -i 时：
//   - UI 从 "Bash" 变成文件编辑样式
//   - 解析 sed 表达式，提取 old/new 对
//   - 显示 diff 预览
```

#### 输出处理

```typescript
// BashTool.tsx
// 大输出 (>30K chars) → 持久化到磁盘文件
if (output.length > LARGE_OUTPUT_THRESHOLD) {
  const filePath = await persistOutput(output)
  return { data: { persistedOutputPath: filePath, preview: output.slice(0, 3000) } }
}

// 图片检测与调整
// 如果输出中发现图片数据 → 自动调整尺寸

// Git index.lock → 特殊错误提示
// 沙箱违规 → stderr 中注释标记
```

#### 自动后台化

```typescript
// 助手模式(KAIROS)下，超过 15 秒的阻塞命令自动转后台
const ASSISTANT_BLOCKING_BUDGET_MS = 15_000
if (isAssistantMode && commandDuration > ASSISTANT_BLOCKING_BUDGET_MS) {
  // 转为异步后台任务
  // 发送 'tengu_bash_command_assistant_auto_backgrounded' 事件
  // 返回 backgroundTaskId
}
```

### 3.6 FileEditTool：原子性读-改-写

`src/tools/FileEditTool/` 包含 6 个文件，~700 行。

#### 验证层（9 道检查）

```
1. 文件大小 (> 1 GiB → 拒绝)
2. Team memory 密钥保护
3. old_string === new_string → 无变更
4. deny 规则匹配
5. 文件存在性 + 空文件/非空内容检查
6. .ipynb 文件 → 引导使用 NotebookEditTool
7. 文件新鲜度（modification time > last read time → "文件已被外部修改"）
8. 字符串匹配（找不到 / 多个匹配 + replace_all=false）
9. settings 文件特殊校验
```

**新鲜度检查的 Windows 兼容性**：Windows 文件时间戳有精度问题，所以在 timestamp 比较失败后还会做内容比对：

```typescript
if (lastWriteTime > lastRead.timestamp) {
  const isFullRead = !lastRead?.offset && !lastRead?.limit
  const contentUnchanged = isFullRead && fileContent === lastRead.content
  if (contentUnchanged) {
    // 时间戳变了但内容没变 → 安全放行
  } else {
    return { result: false, message: 'File modified since read' }
  }
}
```

#### 执行层（原子性 Read-Modify-Write）

```typescript
async call(input, toolUseContext, _, parentMessage) {
  // 1. 技能目录发现（后台、非阻塞）
  const newSkillDirs = await discoverSkillDirsForPaths([absoluteFilePath])
  
  // 2. 确保父目录存在
  await fs.mkdir(dirname(absoluteFilePath), { recursive: true })
  
  // 3. 文件历史追踪（用于 undo）
  if (fileHistoryEnabled()) {
    await fileHistoryTrackEdit(updateFileHistoryState, absoluteFilePath, parentMsg.uuid)
  }
  
  // 4. 临界区：读取 → 新鲜度再验证 → 写入
  const { content: original, encoding, lineEndings } = readFileForEdit(absoluteFilePath)
  
  // 再验证（防止 validate 和 call 之间的竞态）
  const lastWriteTime = getFileModificationTime(absoluteFilePath)
  if (lastWriteTime > lastRead.timestamp && contentChanged) {
    throw new Error(FILE_UNEXPECTEDLY_MODIFIED_ERROR)
  }
  
  // 生成 patch
  const actualOldString = findActualString(original, old_string)
  const actualNewString = preserveQuoteStyle(old_string, actualOldString, new_string)
  const { patch, updatedFile } = getPatchForEdit({
    filePath: absoluteFilePath, fileContents: original,
    oldString: actualOldString, newString: actualNewString, replaceAll: replace_all
  })
  
  // 原子写入（保持原编码和换行符）
  writeTextContent(absoluteFilePath, updatedFile, encoding, lineEndings)
  
  // 5. 后处理
  lspManager?.changeFile(absoluteFilePath, updatedFile)   // 通知 LSP
  lspManager?.saveFile(absoluteFilePath)
  notifyVscodeFileUpdated(absoluteFilePath, original, updatedFile)  // 通知 VS Code
  readFileState.set(absoluteFilePath, { content: updatedFile, timestamp: now })  // 更新缓存
  
  return { data: { filePath, oldString, newString, originalFile, structuredPatch, ... } }
}
```

**`preserveQuoteStyle`**：如果 `old_string` 和 `actualOldString` 之间有引号风格差异（用户可能用了不同的引号），会尝试在 `new_string` 中保持一致。

### 3.7 工具结果回流

工具执行完成后，结果如何回到 Agent Loop：

```typescript
// query.ts — 工具执行后
const toolUpdates = streamingToolExecutor
  ? streamingToolExecutor.getRemainingResults()     // 流式路径
  : runTools(toolUseBlocks, assistantMessages, ...) // 批量路径

for await (const update of toolUpdates) {
  if (update.message) {
    yield update.message  // 发给调用者渲染
    
    // 检查 hook 是否要停止
    if (update.message.type === 'attachment' &&
        update.message.attachment.type === 'hook_stopped_continuation') {
      shouldPreventContinuation = true
    }
    
    // 规范化为 API 可用的 tool_result
    toolResults.push(
      ...normalizeMessagesForAPI([update.message], tools)
        .filter(_ => _.type === 'user'),
    )
  }
  if (update.newContext) {
    updatedToolUseContext = { ...update.newContext, queryTracking }
  }
}
```

**附件注入**（工具结果之后、下一轮 API 调用之前）：

```typescript
// 1. 排队的命令（task-notification, 非斜杠命令）
for await (const attachment of getAttachmentMessages(
  null, updatedToolUseContext, null, queuedCommandsSnapshot,
  [...messagesForQuery, ...assistantMessages, ...toolResults], querySource
)) {
  yield attachment
  toolResults.push(attachment)
}

// 2. 记忆预取（如果已 settle 且未消费）
if (pendingMemoryPrefetch?.settledAt !== null && pendingMemoryPrefetch.consumedOnIteration === -1) {
  const memoryAttachments = filterDuplicateMemoryAttachments(
    await pendingMemoryPrefetch.promise, toolUseContext.readFileState
  )
  for (const memAttachment of memoryAttachments) {
    const msg = createAttachmentMessage(memAttachment)
    yield msg; toolResults.push(msg)
  }
}

// 3. 技能发现预取
if (pendingSkillPrefetch) {
  const skillAttachments = await skillPrefetch.collectSkillDiscoveryPrefetch(pendingSkillPrefetch)
  for (const att of skillAttachments) {
    const msg = createAttachmentMessage(att)
    yield msg; toolResults.push(msg)
  }
}

// 4. MCP 工具刷新
if (updatedToolUseContext.options.refreshTools) {
  const refreshedTools = updatedToolUseContext.options.refreshTools()
  if (refreshedTools !== updatedToolUseContext.options.tools) {
    updatedToolUseContext = { ...updatedToolUseContext, options: { ...options, tools: refreshedTools } }
  }
}
```

最终，所有消息拼接成新的 state 进入下一轮迭代：

```typescript
state = {
  messages: [...messagesForQuery, ...assistantMessages, ...toolResults],
  toolUseContext: toolUseContextWithQueryTracking,
  autoCompactTracking: tracking,
  turnCount: nextTurnCount,
  maxOutputTokensRecoveryCount: 0,
  hasAttemptedReactiveCompact: false,
  pendingToolUseSummary: nextPendingToolUseSummary,
  maxOutputTokensOverride: undefined,
  stopHookActive,
  transition: { reason: 'next_turn' },
}
// → continue → 回到 while(true) 顶部
```

---

---

## 4. 记忆系统：从 MEMORY.md 到自动提取的完整链路

Claude Code 的记忆系统是一个**多层次、多时机、多目录**的持久化知识管理框架。它远不只是"把对话存下来"——而是一个完整的**写入-索引-召回-注入-新鲜度管理**的闭环。

### 4.1 记忆 vs 指令：MEMORY.md 和 CLAUDE.md 的区别

首先要区分两个容易混淆的概念：

| | CLAUDE.md (项目指令) | MEMORY.md (持久记忆) |
|---|---|---|
| **来源** | 人工维护，checked into git | 自动提取 + 用户编辑 |
| **位置** | 项目目录（多级：managed/user/project/local） | `~/.claude/projects/<git-root>/memory/` |
| **加载时机** | 启动时一次，通过 `claudemd.ts` | 说明文本：每次 query 通过 `loadMemoryPrompt()`；索引内容：通过 `getUserContext()` |
| **注入目标** | User context（`getUserContext()`） | 说明文本→ system prompt；索引内容→ user context |
| **目的** | 项目规则和约定 | 跨会话知识积累 |
| **条件加载** | 支持 frontmatter `paths:` 条件 + `@include` | MEMORY.md 索引始终加载；具体记忆文件按需召回 |
| **截断** | 无固定限制（但有 token 预算） | MEMORY.md 索引：200 行 + 25KB；召回单文件：200 行 + 4KB |

### 4.2 记忆目录结构

```
~/.claude/
  └─ projects/
       └─ <canonical-git-root-hash>/
            └─ memory/                 ← getAutoMemPath() 返回这里
                 ├─ MEMORY.md          ← 索引文件（不是记忆本身！）
                 ├─ user_role.md       ← 具体记忆文件（带 frontmatter）
                 ├─ feedback_testing.md
                 ├─ project_context.md
                 ├─ team/              ← 团队记忆目录 (TEAMMEM feature)
                 │    ├─ MEMORY.md
                 │    └─ shared_conventions.md
                 └─ logs/              ← KAIROS 日志目录
                      └─ 2026/
                           └─ 04/
                                └─ 2026-04-11.md
```

**路径解析优先级** (`paths.ts`):

```typescript
// getAutoMemPath() 解析链
1. CLAUDE_COWORK_MEMORY_PATH_OVERRIDE     // Cowork SDK 覆盖
2. settings.json → autoMemoryDirectory    // 用户设置（支持 ~/ 展开）
3. CLAUDE_CODE_REMOTE_MEMORY_DIR          // CCR 远程覆盖
4. ~/.claude/projects/<sanitizedGitRoot>/memory/  // 默认
```

**路径安全验证**：拒绝相对路径、根路径（长度 < 3）、Windows 驱动器根、UNC 路径、null 字节。`sanitizePath()` 对 Git root 做 NFC 标准化 + 特殊字符替换。

### 4.3 记忆文件格式

每个记忆文件是一个 Markdown 文件，带有 frontmatter 头：

```markdown
---
name: User Testing Preferences
description: How the user likes tests written and run
type: feedback
---

The user prefers:
- Jest over Vitest for unit tests
- Integration tests should use docker-compose
- Always run tests before committing
```

**四种记忆类型**（封闭分类法，`memoryTypes.ts`）：

| 类型 | 用途 | 方向 |
|---|---|---|
| `user` | 用户是谁、怎么协作 | → 私有目录 |
| `feedback` | 用户的纠正和偏好 | → 私有目录 |
| `project` | 项目上下文（不可从代码推导的） | → 团队或私有 |
| `reference` | 工具/API 参考文档 | → 团队或私有 |

**不应保存的内容**（`WHAT_NOT_TO_SAVE_SECTION`）：
- 可以从当前代码推导的模式（架构风格、命名约定）
- Git 历史和 commit 风格
- CLAUDE.md 中已有的规则
- 大段代码片段

### 4.4 MEMORY.md 索引文件

MEMORY.md **不是记忆本身，是索引**。每一行是一个指向具体记忆文件的单行指针：

```markdown
- [User Testing Preferences](feedback_testing.md) — Jest over Vitest, docker-compose
- [Deployment Pipeline](project_deploy.md) — Uses GitHub Actions + ArgoCD
```

**约束**：
- 每行不超过 ~150 字符
- 加载时截断到 200 行 + 25KB（`truncateEntrypointContent()`）
- 被截断时会追加警告消息

```typescript
// memdir.ts
export function truncateEntrypointContent(raw: string): EntrypointTruncation {
  const contentLines = raw.trim().split('\n')
  const wasLineTruncated = contentLines.length > MAX_ENTRYPOINT_LINES  // 200
  const wasByteTruncated = raw.trim().length > MAX_ENTRYPOINT_BYTES   // 25,000
  
  // 先按行截断（自然断点），再按字节截断（在最后一个换行符处切断）
  let truncated = wasLineTruncated
    ? contentLines.slice(0, MAX_ENTRYPOINT_LINES).join('\n')
    : raw.trim()
  
  if (truncated.length > MAX_ENTRYPOINT_BYTES) {
    const cutAt = truncated.lastIndexOf('\n', MAX_ENTRYPOINT_BYTES)
    truncated = truncated.slice(0, cutAt > 0 ? cutAt : MAX_ENTRYPOINT_BYTES)
  }
  
  // 追加警告
  return { content: truncated + '\n\n> WARNING: MEMORY.md is ... Only part loaded.' }
}
```

### 4.5 记忆注入：三个时机、三条路径

记忆通过**三条独立的路径**注入到模型的上下文中：

```
┌─────────────────────────────────────────────────────────────────┐
│                     记忆注入全景                                  │
│                                                                 │
│  路径 A：静态注入（每次 query）                                    │
│   ├─ system prompt: loadMemoryPrompt() → 记忆系统使用说明       │
│   └─ user context: getUserContext() → MEMORY.md 索引内容        │
│                                                                 │
│  路径 B：CLAUDE.md 项目指令延迟加载（工具触发）                    │
│   └─ FileReadTool 读文件 → 触发 nestedMemoryAttachmentTriggers   │
│   └─ getNestedMemoryAttachments() → 沿目录树发现 CLAUDE.md       │
│   注意：这不是记忆系统，是 CLAUDE.md 项目指令的延迟加载           │
│                                                                 │
│  路径 C：相关记忆召回（Sonnet 语义匹配）                           │
│   └─ startRelevantMemoryPrefetch() → 异步预取                    │
│   └─ findRelevantMemories() → 扫描 frontmatter + Sonnet 选择    │
│   └─ 直接把具体记忆文件内容插入上下文（不依赖 MEMORY.md）        │
└─────────────────────────────────────────────────────────────────┘
```

#### 路径 A：静态注入（说明 + 索引）

路径 A 通过**两个独立的机制**注入内容：

**机制 1：System Prompt — 记忆系统使用说明**

```typescript
// memdir.ts — loadMemoryPrompt()
// 标准模式 调用 buildMemoryLines()，返回纯指令文本，不含 MEMORY.md 内容
return buildMemoryLines('auto memory', autoDir, extraGuidelines).join('\n')
```

`buildMemoryLines()` 生成的 prompt 包含：
1. **记忆系统描述**：告诉模型有一个持久化文件系统
2. **四种类型定义**（含什么该存什么不该存）
3. **如何保存**：两步流程（写文件 + 更新 MEMORY.md 索引）
4. **何时访问**：收到请求时先检查记忆
5. **信任召回**：引用记忆前先 grep 验证
6. **如何搜索过往上下文**（grep 搜索记忆目录和 transcript 日志）

> 注意：`buildMemoryLines()` **不读取 MEMORY.md 文件内容**。MEMORY.md 索引内容通过下面的机制 2 加载。

**机制 2：User Context — MEMORY.md 索引内容**

```typescript
// context.ts — getUserContext()
// getMemoryFiles() 收集各类 "memory files"（CLAUDE.md + MEMORY.md 等）
// 当 auto memory 启用时，会读取 <autoMemPath>/MEMORY.md （type: 'AutoMem'）
const claudeMd = getClaudeMds(filterInjectedMemoryFiles(await getMemoryFiles()))
```

`getMemoryFiles()` 内部检测到 auto memory 启用时，会读取 MEMORY.md 并标记为 `'AutoMem'` 类型。`filterInjectedMemoryFiles()` 根据 `tengu_moth_copse` flag 决定是否保留：
- flag 关闭（默认）：保留 AutoMem 文件 → **MEMORY.md 索引内容被注入 user context**
- flag 开启：过滤掉 AutoMem 文件 → MEMORY.md 不被加载

加载时同样受 200 行 / 25KB 截断限制（`truncateEntrypointContent()`）。

所以模型看到的是：
- System prompt 中：“你有一个记忆系统在这个路径…”（指令）
- User context 中：“Contents of MEMORY.md (user's auto-memory):…”（索引内容）

模型看到了记忆的**标题列表**，但看不到具体记忆文件的内容——那些由 Path C 或模型主动 FileRead 提供。

#### 路径 B：CLAUDE.md 项目指令延迟加载

> **注意**：路径 B 与 auto-memory（MEMORY.md）系统**无关**。它加载的是 **CLAUDE.md 项目指令**，不是记忆文件。代码中叫“嵌套记忆”是因为 CLAUDE.md 加载和记忆系统共享同一个模块（`claudemd.ts`）。

当模型使用 `FileReadTool` 读取文件时，系统沿着文件路径的目录树向上查找并加载各级 CLAUDE.md：

```typescript
// attachments.ts
async function getNestedMemoryAttachmentsForFile(filePath, toolUseContext, appState) {
  // Phase 1: Managed rules（~/.claude/rules/）
  // Phase 2: User conditional rules
  // Phase 3: 嵌套目录（CWD → 目标文件，每级的 CLAUDE.md + .claude/CLAUDE.md）
  // Phase 4: CWD 级目录（root → CWD，只加载条件规则）
}
```

目的：不同子目录可以有自己的 CLAUDE.md 指令（如 `tests/CLAUDE.md` 说“用 pytest”），只有模型读到这个目录下的文件时才加载，避免启动时一口气加载所有子目录的规则浪费 token。

**两层去重**（防止 LRU 驱逐后重复注入）：

```typescript
if (loadedNestedMemoryPaths.has(path)) skip     // Tier 1: 永久集合（不驱逐）
if (readFileState.has(path)) skip               // Tier 2: 100 项 LRU（可能驱逐）
else:
  loadedNestedMemoryPaths.add(path)             // 加入永久集合
  readFileState.set(path, {...})                // 加入 LRU
```

#### 路径 C：相关记忆召回（Sonnet 语义匹配）

这是最精密的路径。每次用户输入时，异步预取相关记忆：

```typescript
// query.ts — 循环入口
using pendingMemoryPrefetch = startRelevantMemoryPrefetch(state.messages, state.toolUseContext)

// 在工具执行期间（5-30秒），预取并行运行
// 工具完成后消费：
if (pendingMemoryPrefetch?.settledAt !== null && pendingMemoryPrefetch.consumedOnIteration === -1) {
  const memoryAttachments = filterDuplicateMemoryAttachments(
    await pendingMemoryPrefetch.promise, toolUseContext.readFileState
  )
  for (const memAttachment of memoryAttachments) {
    yield createAttachmentMessage(memAttachment)
  }
  pendingMemoryPrefetch.consumedOnIteration = turnCount - 1
}
```

**召回流水线**：

```
用户输入 "为什么部署到 staging 失败了？"
     │
     ├─① startRelevantMemoryPrefetch()
     │   ├─ 检查：auto memory 启用？非单词输入？未超 60*1024 字节会话限额？
     │   └─ 异步启动 getRelevantMemoryAttachments()
     │
     ├─② scanMemoryFiles(memoryDir)
     │   ├─ 递归扫描 *.md（排除 MEMORY.md）
     │   ├─ 读前 30 行提取 frontmatter（name, description, type）
     │   ├─ 按 mtime 降序排序
     │   └─ 最多 200 个文件
     │
     ├─③ selectRelevantMemories() — Sonnet 语义选择
     │   ├─ 构建 manifest: "- [feedback] deploy_issues.md (2026-04-10): staging 部署常见问题"
     │   ├─ 如果最近成功使用了某工具 → 排除该工具的 reference 文档（减噪）
     │   ├─ 调用 Sonnet (sideQuery): "选最多 5 个最相关的记忆"
     │   └─ JSON 输出: { selected_memories: ["deploy_issues.md", "project_deploy.md"] }
     │
     ├─④ readMemoriesForSurfacing()
     │   ├─ 读取选中文件内容
     │   ├─ 每文件最多 200 行 + 4KB
     │   └─ 预计算 header（含新鲜度提醒）
     │
     └─⑤ filterDuplicateMemoryAttachments()
          ├─ 排除已被 FileRead/Edit/Write 触及的文件
          ├─ 标记到 readFileState 防下轮重复
          └─ 作为 AttachmentMessage yield 给模型
```

**会话级节流**：累计召回的记忆字节不超过 `MAX_SESSION_BYTES = 60 * 1024 = 61,440 字节`。通过扫描消息历史中的 `relevant_memories` 附件计算累计量。Compact 后这些附件从历史中消失，计数器自然重置。

**新鲜度管理** (`memoryAge.ts`)：

```typescript
// 超过 1 天的记忆会附带警告
memoryFreshnessText(mtimeMs):
  "This memory is 47 days old. Memories are point-in-time observations,
   not live state — claims about code behavior or file:line citations
   may be outdated. Verify against current code before asserting as fact."
```

### 4.6 自动记忆提取：后台子 Agent

**位置**：`src/services/extractMemories/extractMemories.ts`

这是记忆系统的"写入"端。每次 Agent Loop 的 query 完成（`handleStopHooks()`），会 fire-and-forget 一个后台子 Agent 来分析对话并提取值得记住的知识。

#### 初始化与闭包状态

```typescript
// 闭包封装的可变状态（非模块级，每次 initExtractMemories() 新建）
export function initExtractMemories(): void {
  let lastMemoryMessageUuid: string | undefined    // 游标：上次处理到的消息
  let inProgress = false                           // 互斥锁
  let turnsSinceLastExtraction = 0                 // 节流计数器
  let pendingContext: { context, appendSystemMessage } | undefined  // 合并缓冲
  const inFlightExtractions = new Set<Promise<void>>()  // 优雅关闭追踪
  // ...
}
```

#### 执行条件

```
executeExtractMemories() 的执行门控：
  ├─ 只对主 Agent 执行（agentId 为空）
  ├─ Statsig gate: tengu_passport_quail = true
  ├─ isAutoMemoryEnabled() = true
  ├─ 非远程模式 (getIsRemoteMode() = false)
  ├─ 互斥：如果正在进行中 → 暂存上下文等待尾随运行
  └─ 节流：每 N 个合格 turn 执行一次（tengu_bramble_lintel, 默认 1）
```

#### 互斥与合并

主 Agent 和后台提取 Agent 是**每轮互斥**的：

```typescript
// 互斥检测
if (hasMemoryWritesSince(messages, lastMemoryMessageUuid)) {
  // 主 Agent 已经写了记忆 → 跳过提取，推进游标
  lastMemoryMessageUuid = lastMessage?.uuid
  return
}
```

如果提取正在运行时又被调用（新一轮完成了），不会启动第二个并发提取，而是暂存上下文：

```typescript
if (inProgress) {
  pendingContext = { context, appendSystemMessage }  // 覆盖旧的（只保留最新）
  return
}
```

当前提取完成后，如果有暂存上下文，会启动**尾随运行**。尾随运行的游标已推进到当前提取完成的位置，所以它只处理新增的消息。

#### Forked Agent 模式

提取使用 `runForkedAgent()` — **完美 fork 主对话**，共享 prompt cache：

```typescript
const result = await runForkedAgent({
  promptMessages: [createUserMessage({ content: userPrompt })],
  cacheSafeParams,               // 共享 system prompt + 消息前缀（cache key 一致）
  canUseTool: createAutoMemCanUseTool(memoryDir),
  querySource: 'extract_memories',
  forkLabel: 'extract_memories',
  skipTranscript: true,          // 不写入会话 transcript（避免竞态）
  maxTurns: 5,                   // 硬上限，防止验证死胡同
})
```

#### 提取 Agent 的工具权限

```typescript
export function createAutoMemCanUseTool(memoryDir: string): CanUseToolFn {
  return async (tool, input) => {
    // ✅ 允许：REPL（ant 模式下原始工具被隐藏，REPL 内部再走回 canUseTool）
    // ✅ 允许：FileRead, Grep, Glob — 不限路径（只读）
    // ✅ 允许：Bash — 仅 isReadOnly() 的命令（ls/find/cat/stat/wc/head/tail）
    // ✅ 允许：FileEdit, FileWrite — 仅 memoryDir 内的路径
    // ❌ 拒绝：MCP, Agent, 写操作 Bash, 其他一切
    
    if (tool.name === FILE_EDIT_TOOL_NAME || tool.name === FILE_WRITE_TOOL_NAME) {
      if (typeof input.file_path === 'string' && isAutoMemPath(input.file_path)) {
        return { behavior: 'allow', updatedInput: input }
      }
    }
    return denyAutoMemTool(tool, '...')
  }
}
```

#### 提取 Prompt

```typescript
// prompts.ts — opener()
`You are now acting as the memory extraction subagent.
 Analyze the most recent ~${newMessageCount} messages above
 and use them to update your persistent memory systems.`

// 效率约束：
`You have a limited turn budget.
 FileEdit requires a prior FileRead of the same file,
 so the efficient strategy is:
   turn 1 — issue all FileRead calls in parallel
   turn 2 — issue all FileWrite/FileEdit calls in parallel
 Do not interleave reads and writes across multiple turns.`

// 禁止验证：
`You MUST only use content from the last ~${newMessageCount} messages.
 Do not waste any turns attempting to investigate or verify — 
 no grepping source files, no reading code to confirm a pattern exists.`
```

#### 结果处理

```typescript
// 提取完成后
const writtenPaths = extractWrittenPaths(result.messages)
const memoryPaths = writtenPaths.filter(p => basename(p) !== ENTRYPOINT_NAME)

if (memoryPaths.length > 0) {
  // 在 REPL 中显示 "Memory saved: user_role.md, feedback_testing.md"
  appendSystemMessage?.(createMemorySavedMessage(memoryPaths))
}

// 推进游标
lastMemoryMessageUuid = lastMessage?.uuid
```

#### 优雅关闭

```typescript
// cli/print.ts — 会话退出前
drainPendingExtraction(60_000)  // 等待最多 60 秒让提取完成

// 实现
let drainer = async (timeoutMs = 60_000) => {
  if (inFlightExtractions.size === 0) return
  await Promise.race([
    Promise.all(inFlightExtractions).catch(() => {}),
    new Promise(r => setTimeout(r, timeoutMs).unref()),  // .unref() 不阻塞进程退出
  ])
}
```

### 4.7 记忆扫描与目录管理

**`memoryScan.ts`** — 统一的目录扫描原语：

```typescript
export async function scanMemoryFiles(memoryDir, signal): Promise<MemoryHeader[]> {
  const entries = await readdir(memoryDir, { recursive: true })
  const mdFiles = entries.filter(f => f.endsWith('.md') && basename(f) !== 'MEMORY.md')
  
  // 并行读取每个文件的前 30 行提取 frontmatter
  const headers = await Promise.allSettled(
    mdFiles.map(async (relativePath) => {
      const { content, mtimeMs } = await readFileInRange(filePath, 0, 30, undefined, signal)
      const { frontmatter } = parseFrontmatter(content, filePath)
      return {
        filename: relativePath,
        filePath,
        mtimeMs,
        description: frontmatter.description || null,
        type: parseMemoryType(frontmatter.type),  // user|feedback|project|reference
      }
    })
  )
  
  // 按修改时间降序，最多 200 个
  return headers
    .filter(r => r.status === 'fulfilled')
    .map(r => r.value)
    .sort((a, b) => b.mtimeMs - a.mtimeMs)
    .slice(0, MAX_MEMORY_FILES)
}

// 格式化为 manifest（给 Sonnet selector 和提取 Agent 用）
export function formatMemoryManifest(memories: MemoryHeader[]): string {
  return memories.map(m => {
    const tag = m.type ? `[${m.type}] ` : ''
    const ts = new Date(m.mtimeMs).toISOString()
    return m.description
      ? `- ${tag}${m.filename} (${ts}): ${m.description}`
      : `- ${tag}${m.filename} (${ts})`
  }).join('\n')
}
```

### 4.8 团队记忆 (TEAMMEM)

**Feature-gated**: `feature('TEAMMEM')`

团队记忆是 auto memory 的扩展，在同一个 memory 目录下增加 `team/` 子目录：

```
memory/
  ├─ MEMORY.md          ← 私有索引
  ├─ private files...
  └─ team/
       ├─ MEMORY.md     ← 团队索引
       └─ shared files...
```

**安全验证** (`teamMemPaths.ts`)：

```typescript
export function validateTeamMemWritePath(path: string): boolean {
  // 1. realpathDeepestExisting() — 解析现有部分的真实路径
  // 2. 真实包含性检查（防止符号链接逃逸）
  // 3. 拒绝 null 字节、URL 编码遍历、Unicode 攻击
  // 4. 捕获 ELOOP（符号链接循环）
}
```

### 4.9 KAIROS 模式（长驻助手）

**Feature-gated**: `feature('KAIROS')`

助手会话是实质上永久运行的，所以记忆策略不同：

```typescript
function buildAssistantDailyLogPrompt(): string {
  // 不维护 MEMORY.md 索引，改为追加日志
  // 路径模式：memory/logs/YYYY/MM/YYYY-MM-DD.md
  // 每条记录是带时间戳的 bullet point
  // MEMORY.md 由 nightly /dream skill 从日志蒸馏
  
  return `
    This session is long-lived.
    Record by **appending** to today's daily log file.
    Do not rewrite or reorganize the log — it is append-only.
    A separate nightly process distills logs into MEMORY.md.
  `
}
```

### 4.10 /memory 命令

**位置**：`src/commands/memory/`

```typescript
// 用户交互流程
/memory
  → 打开 MemoryFileSelector 对话框
  → 选择或创建记忆文件
  → 用 $EDITOR / $VISUAL 打开编辑
  → 保存后自动生效（下次 query 重新加载）
```

### 4.11 记忆与 Compact 的交互

Compact（上下文压缩）对记忆系统有以下影响：

1. **`loadedNestedMemoryPaths` 被清除**：compact 后嵌套记忆可以重新注入
2. **`readFileState` 被清除**：compact 调用 `readFileState.clear()`
3. **`collectSurfacedMemories()` 自然重置**：扫描的是压缩后的消息，旧附件已消失
4. **Post-compact 文件恢复**：最多恢复 5 个最近文件（每个 5K token，总 50K token）
5. **MEMORY.md 在 system prompt 中**：不受 compact 影响（每次重新加载）

### 4.12 记忆系统关键常量

| 常量 | 值 | 位置 |
|---|---|---|
| `MAX_ENTRYPOINT_LINES` | 200 | MEMORY.md 行数上限 |
| `MAX_ENTRYPOINT_BYTES` | 25,000 | MEMORY.md 字节上限 |
| `MAX_MEMORY_FILES` | 200 | 扫描的最大文件数 |
| `FRONTMATTER_MAX_LINES` | 30 | frontmatter 扫描行数 |
| `MAX_MEMORY_LINES` | 200 | 召回单文件行数 |
| `MAX_MEMORY_BYTES` | 4,096 | 召回单文件字节（~20KB/轮） |
| `MAX_SESSION_BYTES` | 61,440 (60×1024) | 会话累计召回上限 |
| 选择上限 | 5 | Sonnet 每轮最多选 5 个记忆 |
| `maxTurns` | 5 | 提取 Agent 的 turn 上限 |
| drain timeout | 60,000ms | 优雅关闭等待时间 |

### 4.13 记忆系统完整数据流

```
┌─ 写入端 ──────────────────────────────────────────────────┐
│                                                            │
│  主 Agent 直接写入 ──────────┐                              │
│    └─ 用户说"记住这个"        │                             │
│    └─ FileWrite/Edit 到      ├─→ memory/ 目录               │
│       memory/ 目录            │                             │
│                              │                              │
│  后台提取 Agent ──────────────┘                              │
│    └─ initExtractMemories()                                 │
│    └─ 闭包状态：游标 + 互斥 + 节流                            │
│    └─ runForkedAgent()（共享 prompt cache）                   │
│    └─ 受限工具集：Read/Grep/Glob/只读Bash + 限目录Write       │
│    └─ 2-turn 策略：批量读 → 批量写                            │
│                                                             │
│  /memory 命令 ─────────────────→ 用户手动编辑                 │
│  /dream 技能 ─────────────────→ 日志蒸馏（KAIROS）            │
│                                                             │
└─────────────────────────────────────────────────────────────┘

┌─ 读取端 ──────────────────────────────────────────────────┐
│                                                            │
│  路径 A: System Prompt + User Context                      │
│    loadMemoryPrompt() → 记忆系统使用说明 → system prompt     │
│    getUserContext() → MEMORY.md 索引内容 → user context      │
│                                                            │
│  路径 B: 嵌套记忆                                           │
│    FileRead 触发 → nestedMemoryAttachmentTriggers           │
│    → getNestedMemoryAttachmentsForFile() → CLAUDE.md 发现   │
│    → 两层去重 → AttachmentMessage                           │
│                                                            │
│  路径 C: 相关记忆召回                                        │
│    startRelevantMemoryPrefetch() → 异步                     │
│    → scanMemoryFiles() → formatMemoryManifest()             │
│    → selectRelevantMemories() → Sonnet sideQuery            │
│    → readMemoriesForSurfacing() → 4KB/文件截断              │
│    → filterDuplicateMemoryAttachments() → 去重              │
│    → AttachmentMessage → 含新鲜度提醒                       │
│                                                            │
│  节流：60*1024 字节会话累计上限，compact 后自然重置             │
│  去重：readFileState (LRU) + loadedNestedMemoryPaths (Set)  │
│  新鲜度：>1天 → 附加验证提醒                                 │
│                                                            │
└─────────────────────────────────────────────────────────────┘
```

---

## 附录：关键常量速查

| 常量 | 值 | 用途 |
|---|---|---|
| `AUTOCOMPACT_BUFFER_TOKENS` | 13,000 | autocompact 触发缓冲区 |
| `WARNING_THRESHOLD_BUFFER_TOKENS` | 20,000 | 用户警告阈值 |
| `MANUAL_COMPACT_BUFFER_TOKENS` | 3,000 | 硬阻塞限制 |
| `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES` | 3 | 熔断次数 |
| `MAX_OUTPUT_TOKENS_RECOVERY_LIMIT` | 3 | max_tokens 恢复次数 |
| `ESCALATED_MAX_TOKENS` | 64,000 | 升级后的输出 token 限制 |
| `MAX_FILES_TO_RESTORE` | 5 | compact 后恢复的文件数 |
| 每文件 token 上限 | 5,000 | compact 后单文件限制 |
| 文件总预算 | 50,000 | compact 后文件总限制 |
| Time-based MC gap | 60 min | 冷缓存阈值 |
| Time-based MC keepRecent | 5 | 保留最近 N 个工具结果 |
| BashTool auto-bg budget | 15,000 ms | 助手模式阻塞预算 |
| FileEditTool max file size | 1 GiB | 文件大小上限 |
| BashTool large output | 30,000 chars | 持久化阈值 |
| SM-Compact `minTokens` | 10,000 | 压缩后至少保留的 token 数 |
| SM-Compact `minTextBlockMessages` | 5 | 至少保留的含文本消息数 |
| SM-Compact `maxTokens` | 40,000 | 压缩后保留的 token 硬上限 |
| Session Memory `minimumMessageTokensToInit` | 10,000 | 首次提取的 token 阈值 |
| Session Memory `minimumTokensBetweenUpdate` | 5,000 | 两次提取间的最小 token 增长 |
| Session Memory `toolCallsBetweenUpdates` | 3 | 两次提取间的最小工具调用数 |
| `EXTRACTION_WAIT_TIMEOUT_MS` | 15,000 | SM-Compact 等待提取完成的超时 |
| `EXTRACTION_STALE_THRESHOLD_MS` | 60,000 | 视为僵尸提取的阈值 |
| `MAX_SECTION_LENGTH` | 2,000 | Session Memory 单章节 token 上限 |
| `MAX_TOTAL_SESSION_MEMORY_TOKENS` | 12,000 | Session Memory 总 token 上限 |
