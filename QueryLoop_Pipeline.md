# Claude Code queryLoop 完整执行流程

> 基于 `src/query.ts` 源码逐行梳理，覆盖 `while(true)` 循环内的每一个步骤。

---

## 循环前：一次性初始化

```
queryLoop() 入口
  │
  ├─ 解构不可变参数（systemPrompt, userContext, canUseTool, fallbackModel, maxTurns 等）
  ├─ 初始化可变 State 对象（messages, toolUseContext, turnCount=1, transition=undefined ...）
  ├─ 创建 budgetTracker（TOKEN_BUDGET feature gate）
  ├─ 快照运行时配置 buildQueryConfig()（feature gate、session 级开关等，循环内不再读）
  ├─ 异步启动记忆预取 startRelevantMemoryPrefetch()
  │     └─ 用 `using` 语法确保 generator 退出时自动 dispose
  └─ 进入 while(true)
```

---

## 每次迭代的完整流程

```
┌─────────────────────────────────────────────────────────────────┐
│ while (true) {                                                  │
│                                                                 │
│  ═══════════ 阶段 1：状态解构 + 预取启动 ═══════════            │
│                                                                 │
│  1.  解构 state → messages, toolUseContext, turnCount 等         │
│  2.  启动技能发现预取 startSkillDiscoveryPrefetch()              │
│      （异步，利用后续 API 调用和工具执行时间并行完成）             │
│  3.  yield { type: 'stream_request_start' }                     │
│  4.  初始化/递增 queryTracking（chainId + depth）                │
│  5.  messagesForQuery = 从 compact 边界之后截取消息               │
│                                                                 │
│  ═══════════ 阶段 2：五层压缩流水线 ═══════════                  │
│                                                                 │
│  6.  ① applyToolResultBudget()                                  │
│      扫描所有 tool_result，超出 per-message 预算的持久化到磁盘    │
│      FileRead 等 maxResultSizeChars=Infinity 的工具被跳过         │
│                                                                 │
│  7.  ② snipCompactIfNeeded()                     [HISTORY_SNIP] │
│      从发给 API 的消息列表中成对删除被标记的 tool_use+tool_result │
│      "直接不发"——改变消息序列结构，prompt cache 失效              │
│      触发频率很低（模型主动调 SnipTool 才触发），记录 snipTokensFreed│
│      如有 boundaryMessage 则 yield                               │
│                                                                 │
│  8.  ③ microcompact()                                           │
│      删最旧的 8 类工具 tool_result，保留最近 N 个                 │
│      热缓存 → cache_edits：消息文本不改，API 推理时忽略对应 KV   │
│              "发了但让模型假装没看到"，cache key 不变              │
│      冷缓存 → 物理清空旧 tool_result 为占位符                    │
│      记录 pendingCacheEdits 供 API 响应后出 boundary 消息         │
│                                                                 │
│  9.  ④ applyCollapsesIfNeeded()               [CONTEXT_COLLAPSE]│
│      结构保留的归档压缩，旧轮次 → 独立摘要                       │
│                                                                 │
│  10. 构建完整 system prompt                                      │
│      appendSystemContext(systemPrompt, systemContext)            │
│                                                                 │
│  11. ⑤ autocompact()                                            │
│      ├─ 先尝试 SM-Compact（读 Session Memory 文件，零 API 调用） │
│      ├─ 不可用则 Full Compact（fork 子 Agent 调 API 摘要）       │
│      ├─ 成功 → yield 所有 postCompactMessages                   │
│      │        messagesForQuery = postCompactMessages              │
│      │        重置 tracking（turnCounter=0）                     │
│      └─ 失败 → 递增 consecutiveFailures（熔断器：≥3 次停止）     │
│                                                                 │
│  12. 更新 toolUseContext.messages = messagesForQuery             │
│                                                                 │
│  ═══════════ 阶段 3：预飞检查 ═══════════                        │
│                                                                 │
│  13. 初始化 StreamingToolExecutor（如果启用流式工具执行）         │
│  14. 解析当前模型（考虑 plan mode、200K+ token 切换）             │
│  15. 创建 dumpPromptsFetch（调试用，仅 ant 用户）                │
│  16. 阻塞限制检查（auto-compact 关闭时）                         │
│      ├─ 计算 isAtBlockingLimit                                   │
│      └─ 超限 → yield 错误消息 → return { reason: 'blocking_limit' }│
│                                                                 │
│  ═══════════ 阶段 4：调用 Claude API（流式） ═══════════         │
│                                                                 │
│  17. 构建 API 请求参数                                           │
│      ├─ prependUserContext(messages, userContext)                │
│      ├─ 工具列表、thinking 配置、模型、fallback 等               │
│      ├─ task_budget（如有）                                      │
│      └─ effortValue、advisorModel、skipCacheWrite 等             │
│                                                                 │
│  18. for await (message of callModel(...)) {                    │
│      │                                                           │
│      ├─ 模型回退处理                                             │
│      │   如果流式中途触发 onStreamingFallback:                   │
│      │   → tombstone 已有的 assistantMessages                    │
│      │   → 清空状态，重建 StreamingToolExecutor                  │
│      │                                                           │
│      ├─ backfillObservableInput                                  │
│      │   tool_use block 追加可观测字段（给 SDK/transcript 用）    │
│      │                                                           │
│      ├─ 暂扣可恢复错误（不 yield 给 UI）                         │
│      │   ├─ 413 prompt-too-long → withheld                      │
│      │   ├─ max_output_tokens → withheld                        │
│      │   └─ media_size_error → withheld                         │
│      │   其他正常消息 → yield                                    │
│      │                                                           │
│      ├─ 收集 assistantMessages                                  │
│      ├─ 提取 tool_use blocks → toolUseBlocks, needsFollowUp=true│
│      │                                                           │
│      ├─ 流式工具并行执行（模型还在输出时）                       │
│      │   streamingToolExecutor.addTool(toolBlock)                │
│      │   → 只读工具并行，写操作串行                              │
│      │                                                           │
│      └─ 即时收割已完成的工具结果                                 │
│          streamingToolExecutor.getCompletedResults()              │
│          → yield result.message                                  │
│      }                                                           │
│                                                                 │
│  19. Cached Microcompact 后处理                                  │
│      API 响应中读取 cache_deleted_input_tokens                   │
│      → yield microcompact boundary message（实际删除的 token 数） │
│                                                                 │
│  20. Fallback 重试（catch FallbackTriggeredError）               │
│      → 切换模型，strip thinking signatures，yield 系统通知       │
│      → attemptWithFallback = true → 重新进入 API 调用            │
│                                                                 │
│  21. 异常处理（catch error）                                     │
│      → yield 缺失的 tool_result（防止 tool_use/result 不配对）   │
│      → yield 错误消息 → return { reason: 'model_error' }        │
│                                                                 │
│  ═══════════ 阶段 5：后采样处理 ═══════════                      │
│                                                                 │
│  22. executePostSamplingHooks()（异步 fire-and-forget）          │
│      ├─ Session Memory 提取                                     │
│      ├─ ToolUseSummary 生成（Haiku）                             │
│      └─ 分析事件记录                                             │
│                                                                 │
│  23. 中断检查                                                    │
│      如果 abortController.signal.aborted:                        │
│      ├─ 消费 StreamingToolExecutor 剩余结果（生成合成 tool_result）│
│      ├─ Chicago MCP 清理（computer use 解锁）                    │
│      ├─ yield 中断消息（除非是 submit-interrupt）                │
│      └─ return { reason: 'aborted_streaming' }                  │
│                                                                 │
│  24. yield 上一轮的 ToolUseSummary                               │
│      （Haiku ~1s，在本轮 API 调用 5-30s 期间已并行完成）         │
│                                                                 │
│  ═══════════ 阶段 6：无工具调用时的恢复/退出逻辑 ═══════════     │
│  （needsFollowUp = false 时进入此分支）                          │
│                                                                 │
│  25. 413 恢复链                                                  │
│      ├─ 第一级：Context Collapse drain                           │
│      │   提交所有暂存的 collapse → continue (collapse_drain_retry)│
│      ├─ 第二级：Reactive Compact                                 │
│      │   tryReactiveCompact() → continue (reactive_compact_retry)│
│      └─ 都失败 → yield 暂扣的错误 → return { reason: 'prompt_too_long' }│
│                                                                 │
│  26. media_size_error 恢复                                       │
│      Reactive Compact strip-retry → continue 或 return           │
│                                                                 │
│  27. max_output_tokens 恢复                                      │
│      ├─ 第一步：升级到 64K → continue (max_output_tokens_escalate)│
│      ├─ 第二步：注入续写消息 → continue (max_output_tokens_recovery)│
│      │   "Resume directly — no recap"，最多 3 次                 │
│      └─ 3 次用完 → yield 暂扣的错误                              │
│                                                                 │
│  28. API 错误跳过 stop hooks                                     │
│      如果 lastMessage.isApiErrorMessage:                         │
│      → executeStopFailureHooks() → return { reason: 'completed' }│
│                                                                 │
│  29. Stop Hooks 执行                                             │
│      handleStopHooks() → 依次执行:                               │
│      ├─ Template Job Classification（同步）                      │
│      ├─ Prompt Suggestion（异步 fire-and-forget）                │
│      ├─ Memory Extraction（异步 fire-and-forget）                │
│      ├─ Auto-Dream（异步 fire-and-forget）                       │
│      ├─ Chicago MCP 清理（同步）                                 │
│      └─ 用户注册的 executeStopHooks()                            │
│      │                                                           │
│      ├─ preventContinuation → return { reason: 'stop_hook_prevented' }│
│      └─ blockingErrors → 注入错误消息 → continue (stop_hook_blocking)│
│                                                                 │
│  30. Token Budget 续写检查                  [TOKEN_BUDGET]       │
│      ├─ action='continue' → 注入 nudge 消息                     │
│      │   → continue (token_budget_continuation)                  │
│      ├─ diminishingReturns → 提前停止                            │
│      └─ 正常完成                                                 │
│                                                                 │
│  31. return { reason: 'completed' } ← 正常退出点                 │
│                                                                 │
│  ═══════════ 阶段 7：有工具调用时的后处理 ═══════════             │
│  （needsFollowUp = true 时进入此分支）                           │
│                                                                 │
│  32. 收割剩余工具结果                                            │
│      StreamingToolExecutor.getRemainingResults()                  │
│      或 runTools()（非流式路径的批量执行）                        │
│      → yield 每个 update.message                                 │
│      → 检查 hook_stopped_continuation                            │
│      → 更新 updatedToolUseContext                                │
│                                                                 │
│  33. 异步生成 ToolUseSummary（Haiku）                            │
│      generateToolUseSummary() → 存为 nextPendingToolUseSummary   │
│      （下一轮迭代步骤 24 消费）                                  │
│                                                                 │
│  34. 工具执行期间中断检查                                        │
│      aborted → Chicago MCP 清理 → yield 中断消息                │
│      → return { reason: 'aborted_tools' }                       │
│                                                                 │
│  35. Hook 停止检查                                               │
│      shouldPreventContinuation → return { reason: 'hook_stopped' }│
│                                                                 │
│  36. 递增 autoCompactTracking.turnCounter                        │
│                                                                 │
│  ═══════════ 阶段 8：附件注入 ═══════════                        │
│                                                                 │
│  37. 排队命令快照                                                │
│      getCommandsByMaxPriority()                                  │
│      ├─ 过滤掉 slash 命令                                       │
│      ├─ 主线程只拿 agentId=undefined 的                          │
│      └─ 子 Agent 只拿自己 agentId 的 task-notification           │
│                                                                 │
│  38. getAttachmentMessages()                                     │
│      ├─ 文件变更附件（edited_text_file）                         │
│      ├─ 排队命令转为附件                                         │
│      └─ 其他上下文附件                                           │
│      → yield 每个附件 → 追加到 toolResults                       │
│                                                                 │
│  39. 记忆预取消费                                                │
│      如果 pendingMemoryPrefetch 已 settle 且未消费:              │
│      → filterDuplicateMemoryAttachments()（排除已 Read 的文件）  │
│      → yield 每个记忆附件 → 追加到 toolResults                   │
│      → 标记 consumedOnIteration                                  │
│                                                                 │
│  40. 技能发现预取消费                                            │
│      collectSkillDiscoveryPrefetch()                              │
│      → yield 每个技能附件 → 追加到 toolResults                   │
│                                                                 │
│  41. 已消费命令清理                                              │
│      removeFromQueue() + notifyCommandLifecycle('started')       │
│                                                                 │
│  ═══════════ 阶段 9：MCP 工具刷新 ═══════════                    │
│                                                                 │
│  42. refreshTools()                                              │
│      如果新连接的 MCP Server 带来了新工具 → 更新工具池            │
│                                                                 │
│  ═══════════ 阶段 10：后台任务 + 限制检查 ═══════════            │
│                                                                 │
│  43. 后台任务摘要生成                        [BG_SESSIONS]       │
│      maybeGenerateTaskSummary()（供 `claude ps` 显示进度）        │
│                                                                 │
│  44. maxTurns 检查                                               │
│      nextTurnCount > maxTurns                                    │
│      → yield max_turns_reached 附件                              │
│      → return { reason: 'max_turns' }                           │
│                                                                 │
│  ═══════════ 阶段 11：组装下一轮状态 → continue ═══════════      │
│                                                                 │
│  45. state = {                                                   │
│        messages: [...messagesForQuery, ...assistantMessages,     │
│                   ...toolResults],                               │
│        toolUseContext: toolUseContextWithQueryTracking,           │
│        autoCompactTracking: tracking,                            │
│        turnCount: nextTurnCount,                                 │
│        maxOutputTokensRecoveryCount: 0,      ← 重置             │
│        hasAttemptedReactiveCompact: false,    ← 重置             │
│        pendingToolUseSummary: nextPendingToolUseSummary,          │
│        maxOutputTokensOverride: undefined,   ← 重置             │
│        stopHookActive,                                           │
│        transition: { reason: 'next_turn' },                     │
│      }                                                           │
│      continue → 回到步骤 1                                       │
│                                                                 │
│ } // while (true)                                                │
└─────────────────────────────────────────────────────────────────┘
```

---

## 7 种 continue 路径汇总

| # | transition.reason | 触发条件 | 行为 |
|---|---|---|---|
| 1 | `next_turn` | 正常工具调用完成 | 消息追加工具结果，turnCount++ |
| 2 | `collapse_drain_retry` | 413 + Context Collapse 有暂存 | 提交所有 collapse，重试 API |
| 3 | `reactive_compact_retry` | 413 或 media 过大 | 压缩后重试 |
| 4 | `max_output_tokens_escalate` | 输出 token 限制命中（默认 8K） | 升级到 64K 重试（同一请求） |
| 5 | `max_output_tokens_recovery` | 64K 也不够（最多 3 次） | 注入续写消息 |
| 6 | `stop_hook_blocking` | stop hook 返回阻塞错误 | 注入错误消息让模型修复 |
| 7 | `token_budget_continuation` | token budget 未用完 | 注入 nudge 消息继续生成 |

---

## 8 种退出路径汇总

| return reason | 触发条件 |
|---|---|
| `completed` | 正常完成（无工具调用 + stop hooks 通过 + budget 用完/停止） |
| `blocking_limit` | auto-compact 关闭 + token 到硬限制 |
| `prompt_too_long` | 413 三级恢复全部失败 |
| `image_error` | 图片/媒体错误恢复失败 |
| `model_error` | API 调用抛异常 |
| `aborted_streaming` | 用户 Ctrl+C（流式阶段） |
| `aborted_tools` | 用户 Ctrl+C（工具执行阶段） |
| `max_turns` | 超过 maxTurns 限制 |
| `hook_stopped` | pre-tool hook 的 preventContinuation |
| `stop_hook_prevented` | stop hook 的 preventContinuation |

---

## 关键异步并行时序图

```
时间轴 ──────────────────────────────────────────────────────────►

用户输入
  │
  ├─ 记忆预取启动 ─────────────────────────────── 记忆预取完成 ─┐
  │                                                              │
  ├─ 步骤 6-11: 压缩流水线 ──┐                                  │
  │                           │                                  │
  │                      步骤 17-18: API 调用 ─────────┐         │
  │                           │                        │         │
  │                 技能预取启动 ─────── 技能预取完成 ─┐ │         │
  │                           │                      │ │         │
  │                           │    模型流式输出时:    │ │         │
  │                           │    流式工具并行执行 ──┤ │         │
  │                           │                      │ │         │
  │                           │    API 完成 ─────────┘ │         │
  │                           │                        │         │
  │                      步骤 22: postSamplingHooks ────┤         │
  │                      (fire-and-forget)             │         │
  │                           │                        │         │
  │                      步骤 32: 收割剩余工具结果     │         │
  │                           │                        │         │
  │                      步骤 33: ToolUseSummary ───────┤         │
  │                      (异步, 下轮消费)              │         │
  │                           │                        │         │
  │                      步骤 38: 附件注入             │         │
  │                      步骤 39: 消费记忆预取 ◄───────┘─────────┘
  │                      步骤 40: 消费技能预取 ◄───────┘
  │                           │
  │                      步骤 45: state = next → continue
```

---

## 每轮迭代重置 vs 保持的状态

| 字段 | 正常 continue (next_turn) | 恢复 continue |
|---|---|---|
| `messages` | 追加 assistant + toolResults | 视路径：压缩后替换 / 注入恢复消息 |
| `turnCount` | +1 | 不变（恢复不算新 turn） |
| `maxOutputTokensRecoveryCount` | 重置为 0 | escalate/recovery 时递增 |
| `hasAttemptedReactiveCompact` | 重置为 false | reactive 后设为 true |
| `maxOutputTokensOverride` | 重置为 undefined | escalate 时设为 64K |
| `pendingToolUseSummary` | 设为下一轮的 promise | 重置为 undefined |
| `stopHookActive` | 保持 | blocking 时设为 true |
| `autoCompactTracking` | 保持（turnCounter++） | compact 后重置 |
