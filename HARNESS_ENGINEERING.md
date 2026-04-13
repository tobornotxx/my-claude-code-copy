# Claude Code Harness Engineering 技巧全面总结

> 从源码架构中提炼的 Agent 工程化控制技术。Harness Engineering 指的是在 AI Agent 系统中用于**约束、引导、优化和保护** Agent 行为的工程手段。

---

## 目录

1. [循环控制与状态机设计](#1-循环控制与状态机设计)
2. [上下文预算管理：五层渐进式压缩流水线](#2-上下文预算管理五层渐进式压缩流水线)
3. [工具系统的约束与编排](#3-工具系统的约束与编排)
4. [错误恢复与韧性设计](#4-错误恢复与韧性设计)
5. [权限与安全 Harness](#5-权限与安全-harness)
6. [记忆系统的写入-召回闭环](#6-记忆系统的写入-召回闭环)
7. [性能 Harness：缓存与并行](#7-性能-harness缓存与并行)
8. [生命周期钩子体系](#8-生命周期钩子体系)
9. [资源节流与熔断](#9-资源节流与熔断)
10. [Prompt Engineering Harness](#10-prompt-engineering-harness)
11. [总结：设计原则提炼](#11-总结设计原则提炼)

---

## 1. 循环控制与状态机设计

### 1.1 集中式可变状态 + 不可变参数分离

Agent Loop（`queryLoop`）将运行时状态集中到一个 `State` 对象中，每次 `continue` 时**整体替换**而非逐字段赋值，降低状态不一致的风险：

```typescript
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  turnCount: number
  transition: Continue | undefined  // 记录上一次为什么 continue
  // ... 其他字段
}
```

而不可变的配置参数（systemPrompt、canUseTool、fallbackModel 等）在循环外一次性解构，循环内不修改。

**Harness 意义**：将"会变的"和"不会变的"严格分离，避免循环中无意修改配置导致行为漂移。

### 1.2 Transition 标记：防死循环的显式状态机

`state.transition` 记录了**为什么做了 continue**，而不只是"要不要 continue"。这使得恢复逻辑可以检查上一轮的 transition 原因，避免陷入恢复死循环：

| transition.reason | 含义 |
|---|---|
| `'next_turn'` | 正常工具调用完成 |
| `'collapse_drain_retry'` | 413 后提交 collapse 重试 |
| `'reactive_compact_retry'` | 压缩后重试 |
| `'max_output_tokens_escalate'` | 升级输出 token 限制 |
| `'max_output_tokens_recovery'` | 注入续写消息 |
| `'stop_hook_blocking'` | stop hook 返回错误 |
| `'token_budget_continuation'` | 预算未用完，续写 |

例如：collapse drain 后重试仍然 413 时，检查 `transition.reason === 'collapse_drain_retry'` 就不再走 drain 路径，而是降级到 reactive compact。

**Harness 意义**：显式状态机替代隐式条件嵌套，每种恢复路径有明确的入口和退出条件，可审计、可测试。

### 1.3 AsyncGenerator 作为流式控制原语

`queryLoop` 是一个 `AsyncGenerator`，通过 `yield` 向外部逐条发射消息。调用方（QueryEngine/REPL）逐个消费并渲染。

**Harness 意义**：
- **背压控制**：消费方控制消费速度，生产方不会无限堆积
- **懒求值**：没有消费就不会继续执行，天然支持暂停/取消
- **关注点分离**：循环只管"产出什么"，不管"怎么渲染"

---

## 2. 上下文预算管理：五层渐进式压缩流水线

这是 Claude Code 最精密的 Harness 设计——**不是一刀切压缩，而是五层机制按代价递增顺序依次执行**，前面能搞定就不触发后面：

```
请求进入 → ① Tool Result Budget → ② History Snip → ③ Microcompact → ④ Context Collapse → ⑤ Autocompact → 发给 API
```

### 2.1 第一层：Tool Result Budget（零成本裁切）

- **做什么**：单个工具结果超过 `maxResultSizeChars` 时，持久化到磁盘文件，上下文中只保留路径引用
- **代价**：零（不调 API，不改结构）
- **特殊处理**：`FileRead` 的 `maxResultSizeChars = Infinity`——因为持久化后如果模型再次引用，会触发 Read→file→Read 的死循环

**Harness 技巧**：工具级别的结果预算，在信息进入上下文之前就截断，是最早、最廉价的防线。

### 2.2 第二层：History Snip（精确删除噪声）

- **做什么**：模型可通过 `SnipTool` 标记某些消息为噪声，`snipCompactIfNeeded()` 在每轮循环开头直接删除被标记的消息
- **代价**：零（直接删，不摘要）
- **节流设计**：每增长约 10K tokens 且模型没有主动 snip，会注入 `context_efficiency` 附件消息提醒模型使用 SnipTool
- **与后续层的联动**：`snipTokensFreed` 传给 autocompact 阈值计算，避免 snip 已释放足够空间后误触发 autocompact

**Harness 技巧**：把"什么该删"的决策权交给模型自身（它最知道哪些搜索结果没用），但用注入消息的方式**提醒**它行使这个权利。

### 2.3 第三层：Microcompact（缓存感知的定向删除）

根据缓存状态自动选择路径：

**热缓存路径**（Cached Microcompact）：
- 利用 API 的 `cache_edits` 能力，在**不改消息内容**的情况下告诉 API "删掉这些 token 的缓存"
- 消息内容不变 → prompt cache key 不变 → 其余前缀的 KV cache 完全复用
- 只删除特定工具类型（FileRead, Shell, Grep, Glob, WebSearch, WebFetch, FileEdit, FileWrite）的旧结果缓存

**冷缓存路径**（Time-Based，距上次 assistant 消息 >60 分钟）：
- KV cache 已失效，直接修改消息内容无额外成本
- 保留最近 N 个（默认 5）可压缩的工具结果，其余清空为占位符

**Harness 技巧**：**缓存感知的压缩策略**——同样是"删旧工具结果"的操作，根据缓存是否存活选择不同的实现路径，热缓存时保护 cache key 不被破坏，冷缓存时直接物理删除。

### 2.4 第四层：Context Collapse（结构保留的归档）

- **做什么**：把旧的对话轮次归档为摘要，但**保留轮次结构**（类似 git log，每一轮独立总结）
- **与 autocompact 的区别**：不把所有历史混成一坨
- **恢复功能**：API 返回 413 时有 `recoverFromOverflow()` 方法，提交所有暂存的 collapse

**Harness 技巧**：**渐进式信息降级**——不是"全有或全无"，而是把对话历史从"完整原始消息"逐步降级为"结构化摘要"再到"全量压缩"，每一步都尽可能保留更多结构化信息。

### 2.5 第五层：Autocompact（全量摘要兜底）

两条路径，优先尝试低成本的 SM-Compact：

**SM-Compact（零 API 调用的压缩）**：
- 读取后台 Session Memory 子 Agent 已经维护好的笔记文件作为摘要
- 保留尾部 10K-40K tokens 的原始消息窗口
- 压缩时刻不调 API，毫秒级完成

**Full Compact（传统的 API 压缩）**：
- Fork 子 Agent 调 API 对全部历史做摘要
- 9 章节结构化输出
- 5-10 秒以上耗时

**Harness 技巧**：**成本前移**——SM-Compact 把"生成摘要"的 API 成本从压缩时刻分散到日常后台维护中。压缩触发时只是读取已有文件，延迟从秒级降到毫秒级。

### 2.6 被动防线：Reactive Compact

- **做什么**：不主动压缩，等 API 真的返回 413 再处理
- **与 autocompact 的关系**：启用时**抑制** autocompact 的主动触发
- **暂扣机制**：流式循环中检测到 413 但不立即 yield 给 UI，先尝试恢复；恢复失败才暴露错误

**Harness 技巧**：**乐观执行 + 被动恢复**——不花资源预防可能不会发生的问题，直到真正出错再处理。通过"暂扣错误消息"实现对用户透明的恢复。

### 2.7 流水线设计模式总结

| 层级 | 触发条件 | 代价 | 信息损失 |
|---|---|---|---|
| ① Tool Result Budget | 单工具结果超限 | 零 | 极低（磁盘有完整副本） |
| ② History Snip | 模型标记 + 自动检测 | 零 | 低（删的是模型认定的噪声） |
| ③ Microcompact | 自动（每轮） | 零（热）/ 低（冷） | 中低（删旧工具结果） |
| ④ Context Collapse | token 接近上限 | 中（可能调 API） | 中（保留结构） |
| ⑤ Autocompact | token 超过阈值 | 高（调 API） | 高（全量压缩） |
| ⑥ Reactive Compact | API 返回 413 | 高 | 高 |

**核心原则**：代价递增、信息损失递增、触发频率递减。能用便宜的解决就不用贵的。

---

## 3. 工具系统的约束与编排

### 3.1 Fail-Closed 默认值

`buildTool()` 的默认值全部选择**最保守**的选项：

```typescript
const TOOL_DEFAULTS = {
  isConcurrencySafe: () => false,   // 默认不允许并发
  isReadOnly: () => false,          // 默认假设会写
  isDestructive: () => false,       // 默认非破坏性
  checkPermissions: (input) => Promise.resolve({ behavior: 'allow', updatedInput: input }),
}
```

新工具如果忘了设置，不会被意外并发执行、不会被当成只读操作跳过权限检查。必须**主动声明**才能放宽。

**Harness 意义**：安全约束的默认方向是"收紧"，而不是"放开"。开发者必须显式 opt-in 才能获得更宽松的行为。

### 3.2 读写分类驱动的并发控制

只用一个布尔标记 `isConcurrencySafe` 就实现了完整的并发策略：

- 只读工具之间：可以并行
- 只读 + 写操作：必须等写操作完成
- 写操作 + 写操作：严格串行

```typescript
canExecuteTool(isConcurrencySafe: boolean): boolean {
  const executing = this.tools.filter(t => t.status === 'executing')
  return executing.length === 0 ||
    (isConcurrencySafe && executing.every(t => t.isConcurrencySafe))
}
```

**Harness 意义**：不需要复杂的锁机制或事务系统，一个简单的二分类就足以实现安全的并发控制。

### 3.3 错误级联的精确隔离

Bash 命令出错时取消所有并行兄弟工具（因为 bash 命令间有隐式依赖），但**不取消主循环**：

```typescript
// 用独立的 siblingAbortController，不影响 toolUseContext.abortController
this.siblingAbortController.abort('sibling_error')
```

其他工具（FileRead、WebFetch）出错不级联，因为它们是独立的。

**Harness 意义**：错误影响范围可控——"一个工具出错"不等于"整个 Agent 崩溃"，而是精确地取消受影响的范围。

### 3.4 Deferred Loading：按需加载工具 Schema

40+ 工具的 schema 如果全部发给 API，会占用大量 prompt token。解决方案：

1. 核心工具（Bash/FileRead/FileEdit 等）**始终加载**
2. 低频工具标记 `shouldDefer: true`，schema **不发给模型**
3. 通过 `deferred_tools_delta` 附件告诉模型"有哪些隐藏工具的名字"
4. 模型需要时调用 `ToolSearch` → 拿到完整 schema → 再正常调用

**ToolSearch 的搜索机制**（不是向量搜索，是加权关键词打分）：
- 工具名拆词完全匹配：+10~12 分
- 工具名拆词部分包含：+5~6 分
- `searchHint` 字段匹配：+4 分
- description 词边界匹配：+2 分
- 支持必需词过滤（`+slack send`）

**Harness 意义**：工具能力的"两层索引"——轻量索引（名字列表）始终可见，重量级内容（完整 schema）按需加载。和数据库索引的思路一样。

### 3.5 工具池排序稳定性

所有工具合并后**按名称排序**再发给 API：

```typescript
return uniqBy([...builtInTools].sort(byName).concat(mcpTools.sort(byName)), 'name')
```

**Harness 意义**：保证 prompt cache key 的稳定性。如果工具顺序每次不同（比如 MCP 工具注册顺序不确定），API 会认为是不同的 prompt，缓存全废。

### 3.6 工具结果的序列化-反序列化对称性

每个工具自己定义 `mapToolResultToToolResultBlockParam()`（给 API 的格式）和 `renderToolResultMessage()`（给 UI 的格式），保证：
- API 看到的是最优化的文本表示
- 用户看到的是可读的 React 组件
- 压缩时用的是 `getToolUseSummary()` 生成的极简摘要

**Harness 意义**：同一份工具结果在不同消费场景有不同的表示形式，每种表示都针对其消费方优化。

---

## 4. 错误恢复与韧性设计

### 4.1 暂扣-恢复模式

流式接收 API 响应时，检测到可恢复错误（413、max_output_tokens、media_size）不立即 yield 给 UI，而是"暂扣"：

```typescript
let withheld = false
if (reactiveCompact?.isWithheldPromptTooLong(message)) withheld = true
if (isWithheldMaxOutputTokens(message)) withheld = true
if (!withheld) yield yieldMessage
```

暂扣后进入恢复路径：
- 恢复成功 → 用户完全无感知
- 恢复失败 → 才把错误消息 yield 出去

**Harness 意义**：API 层面的错误不一定是终端用户层面的错误。在"报错"之前先尝试自愈。

### 4.2 Max Output Tokens 渐进升级

模型输出命中 token 限制时，不是直接报错，而是渐进恢复：

1. **第一步**：从默认 8K 升级到 64K (`max_output_tokens_escalate`)
2. **第二步**：64K 还不够，注入 "Resume directly — no recap" 消息让模型续写 (`max_output_tokens_recovery`)
3. **硬上限**：最多恢复 3 次（`maxOutputTokensRecoveryCount`），防止无限续写

**Harness 意义**：输出限制不是断崖式的失败，而是一个多级恢复的渐进过程。

### 4.3 413 的三级恢复链

API 返回 prompt-too-long 时：

1. **Context Collapse drain**：提交所有暂存的 collapse → 重试
2. **Reactive Compact**：触发压缩 → 重试
3. **报错**：两级都失败才报错给用户

**Harness 意义**：使用 transition 标记防止在恢复链中循环（drain 后重试仍 413 → 不再 drain，走 reactive）。

### 4.4 PTL 重试（Prompt-Too-Long Retry）

即使是 compact 子 Agent 本身的摘要请求也可能 413（对话太长连摘要请求都装不下）。解决方案：截断头部消息重试，最多 3 次。

**Harness 意义**：恢复机制自身也需要有兜底恢复策略——"压缩失败"本身需要被处理。

---

## 5. 权限与安全 Harness

### 5.1 六层工具执行检查链

一个工具从模型输出到实际执行要过 6 道关：

```
1. 工具查找（支持别名回退）
2. Zod schema 类型校验（deferred 工具未加载 schema 时提示用 ToolSearch）
3. 自定义业务校验（validateInput）
4. Pre-Hook（外部钩子，可拦截/修改输入）
5. 权限决策（hook 决定 / 用户确认 / 规则匹配）
6. 实际执行（带 OpenTelemetry tracing）
```

**Harness 意义**：多层防御（Defense in Depth），每一层关注不同维度的安全性。

### 5.2 权限来源追踪

每次权限决策都记录来源，方便事后审计：

| 来源 | 分类 |
|---|---|
| Hook 直接批准 | `hook` |
| 会话级临时规则 | `user_temporary` |
| 本地/用户设置规则 | `user_permanent` |
| 用户交互确认 | 使用 tool 返回的 `decisionClassification` |
| 默认配置 | `config` |

**Harness 意义**："这个操作是谁放行的"永远可回溯。

### 5.3 BashTool 的沙箱隔离

不只是"执行命令"，而是：
- 命令解析 + 安全分类（简单/复合/畸形）
- macOS 用 `sandbox-exec`，Linux 用 `seccomp`
- 读写分类决定并发安全性
- `sed -i` 特殊处理（UI 变成编辑预览模式）
- 助手模式下 >15s 命令自动转后台

**Harness 意义**：对最危险的工具（Shell 执行）施加最严格的多维约束。

### 5.4 FileEditTool 的原子性新鲜度检查

防止并发编辑导致文件损坏：
- `validateInput` 阶段：检查文件修改时间
- `call` 阶段执行前：**再检查一次**（防止两阶段之间文件被修改）
- Windows 时间戳精度问题：时间戳对比失败后做**内容对比**兜底
- 写入后通知 LSP 和 VS Code 同步状态

**Harness 意义**：TOCTOU（Time-of-Check to Time-of-Use）防护——validate 和 execute 之间存在时间窗口，必须在 execute 时重新验证。

### 5.5 子 Agent 的工具权限收窄

不同场景的子 Agent 拥有不同的受限工具集：

- **记忆提取 Agent**：只读工具不限路径 + 写操作仅限 memory/ 目录
- **Compact Agent**：只有 FileRead + ToolSearch
- **Coordinator Worker**：文件工具限定目录范围 + 无 MCP

**Harness 意义**：最小权限原则——子 Agent 只获得完成其任务所需的最小工具集。

---

## 6. 记忆系统的写入-召回闭环

### 6.1 索引-内容分离

```
MEMORY.md（索引文件，~25KB 上限）
  → 每行是指针 + 一句话摘要
  → 始终加载到 user context

具体记忆文件（content，每个 ~4KB 上限）
  → 按需通过 Path C 语义召回
  → 或模型主动 FileRead
```

**Harness 意义**：和数据库索引一样——不需要把全部数据塞进上下文，只需要一个足够小的索引来决定"需要加载哪些"。

### 6.2 三条召回路径的分工

| 路径 | 注入什么 | 粒度 | 成本 |
|---|---|---|---|
| **Path A** | 系统说明 + MEMORY.md 索引 | 粗（标题列表） | 零 |
| **Path B** | 子目录 CLAUDE.md 指令 | 中（目录级） | 零（文件读取） |
| **Path C** | 具体记忆文件内容 | 细（文件级，Sonnet 语义选择） | 中（调 Sonnet） |

**Harness 意义**：三层召回粒度——越精细的路径成本越高但精度也越高。粗粒度始终可用作兜底。

### 6.3 异步预取 + 并行利用等待时间

记忆召回的时序设计：

```
用户输入 → 立刻异步启动 Path C 预取
             ↓
         模型开始输出 + 工具执行（5-30 秒）
             ↓ （预取在此期间并行完成）
         工具完成 → 消费预取结果（零额外延迟）
```

**Harness 意义**：不是"需要记忆 → 停下来去取 → 拿到后继续"，而是利用工具执行的自然等待时间并行完成预取。用户感知延迟为零。

### 6.4 写入端的互斥与合并

防止主 Agent 和后台提取 Agent 冲突：
- **互斥检测**：如果主 Agent 这轮已经写了记忆，后台提取跳过
- **并发合并**：提取进行中又来新请求 → 暂存上下文 → 当前完成后尾随运行
- **游标推进**：通过 `lastMemoryMessageUuid` 追踪处理进度，避免重复处理

**Harness 意义**：异步系统中的写冲突管理——不加锁（太重），而是用互斥检测 + 合并缓冲实现轻量级协调。

### 6.5 新鲜度管理

记忆不是写了就永远可信：

```typescript
// 超过 1 天的记忆附带验证提醒
"This memory is 47 days old. Verify against current code before asserting as fact."
```

**Harness 意义**：承认"记忆会过时"的现实。不阻止使用旧记忆，但提醒模型验证后再引用。

### 6.6 会话级节流

累计召回字节不超过 60KB。Compact 后旧附件消失，计数器自然重置。

**Harness 意义**：防止记忆召回本身成为上下文膨胀的来源。通过与 compact 自然联动实现重置，不需要额外管理。

---

## 7. 性能 Harness：缓存与并行

### 7.1 Prompt Cache 稳定性工程

多处设计都是为了保护 prompt cache：

- **工具排序**：按名称排序保证每次请求的工具列表顺序一致
- **Cached Microcompact**：用 `cache_edits` 而非修改消息内容，避免破坏 cache key
- **System prompt 稳定前缀**：`cache_control` 设置确保系统消息作为缓存前缀

**Harness 意义**：Prompt cache 的命中率直接影响 API 调用成本和延迟。所有可能破坏 cache key 的操作都需要显式考虑。

### 7.2 Forked Agent 的缓存共享

子 Agent（记忆提取、compact 摘要）使用 `runForkedAgent()` 而非独立的 API 调用：

```typescript
const result = await runForkedAgent({
  cacheSafeParams,  // 共享 system prompt + 消息前缀 → cache key 一致
  // ...
})
```

因为子 Agent 的请求与主对话共享 prompt 前缀，API 端可以复用主对话的 KV cache。

**Harness 意义**：子 Agent 不是"额外的成本"，通过共享缓存可以大幅降低增量开销。

### 7.3 流式工具执行

模型还在流式输出时就开始执行已收到的工具调用，不等模型说完：

```
模型流式输出: [text...] [tool_use_1] [text...] [tool_use_2] [text...]
                         ↓ 立刻加入队列
                    [tool_use_1 开始执行]
                                        ↓ 立刻加入队列
                                   [tool_use_2 排队/并行]
```

**结果 yield 顺序保证**：即使后面的工具先完成，也按接收顺序输出结果，不会乱序。

**Harness 意义**：流水线式执行，最小化端到端延迟。类似 CPU 的指令流水线——不等一条指令完全完成就开始下一条。

### 7.4 多路异步预取

循环入口处并行启动多个预取任务：

```typescript
// 记忆预取
using pendingMemoryPrefetch = startRelevantMemoryPrefetch(...)
// 技能发现预取
const pendingSkillPrefetch = startSkillDiscoveryPrefetch(...)
```

在工具执行期间并行完成，工具完成后直接消费结果。

**Harness 意义**：利用 Agent 自然的"等待时间"（API 调用、工具执行）并行做预备工作。

---

## 8. 生命周期钩子体系

### 8.1 Stop Hooks：Turn 结束后的生命周期

模型回复完毕（无工具调用）时触发的钩子链：

```
1. Template Job Classification（同步等待）
2. Prompt Suggestion（异步 fire-and-forget）
3. Memory Extraction（异步 fire-and-forget）
4. Auto-Dream（异步 fire-and-forget）
5. Chicago MCP（同步等待）
6. 用户注册的 executeStopHooks()
```

**关键设计**：如果 stop hook 返回 `blockingErrors`，Agent Loop 会把错误注入消息并 `continue`，让模型修复。

**Harness 意义**：在"模型认为已完成"和"系统确认已完成"之间插入验证层。模型说完了不等于真的完了——验证器可能发现问题。

### 8.2 Pre/Post Tool-Use Hooks

工具执行前后的外部钩子：
- **Pre-hook**：可以修改输入（`hookUpdatedInput`）或直接拒绝（`hookPermissionResult`）
- **Post-hook**：可以修改输出（`updatedMCPToolOutput`）

**Harness 意义**：开放的扩展点，允许外部系统在不修改工具代码的情况下拦截/修改工具行为。

### 8.3 Post-Sampling Hooks

每次 API 调用完成后触发，用于非阻塞的后台任务：
- Session Memory 提取
- ToolUseSummary 生成（Haiku）
- 分析事件记录

**Harness 意义**：把不影响主流程的工作推到后台，不阻塞用户交互。

### 8.4 Session Start Hooks

在 compact 之后以 `'compact'` 触发器重新执行，确保压缩后的上下文仍然包含必要的初始化信息。

---

## 9. 资源节流与熔断

### 9.1 Autocompact 熔断器

连续失败 3 次后停止尝试：

```typescript
if (consecutiveFailures >= MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES) {
  logForDebugging('Autocompact circuit breaker tripped')
  return { wasCompacted: false }
}
```

**Harness 意义**：经典的熔断器模式——如果一个操作持续失败，停止尝试避免浪费资源，给系统恢复时间。

### 9.2 记忆提取的多级节流

```
门控 1：只对主 Agent 执行（子 Agent 不提取）
门控 2：Feature gate 必须开启
门控 3：Auto memory 必须启用
门控 4：非远程模式
门控 5：互斥锁（正在提取中则合并）
门控 6：Turn 计数器（每 N 个合格 turn 才执行一次）
```

**Harness 意义**：多级节流确保后台任务不会过度消耗资源。每一层都在不同维度过滤。

### 9.3 Session Memory 提取的双阈值触发

不是每轮都提取，必须同时满足：
- Token 增长 ≥ 5K（有足够新内容）
- 工具调用 ≥ 3 次（有实质性工作发生）

**或者**：Token 增长 ≥ 5K + 最后一轮没有工具调用（自然对话断点）。

**Harness 意义**：双阈值防止在"用户问了一句简单问题"或"只是闲聊"时浪费 API 调用做提取。

### 9.4 SM-Compact 的安全阀

压缩后如果 token 数仍然超过 autocompact 阈值 → 放弃 SM-Compact，回退到 Full Compact。

```typescript
if (postCompactTokenCount >= autoCompactThreshold) return null  // 回退
```

**Harness 意义**：快速路径有失败兜底。SM-Compact 是乐观路径（假设 Session Memory 足够好），但如果效果不够就自动降级。

### 9.5 SM-Compact 等待提取完成的超时机制

```typescript
// 等待后台 Session Memory 提取完成，最多 15 秒
await waitForSessionMemoryExtraction()
// 超过 60 秒的提取视为僵尸进程，不再等待
```

**Harness 意义**：异步依赖不能无限等待。合理的超时 + 僵尸检测确保系统不会因为后台任务卡住而阻塞。

---

## 10. Prompt Engineering Harness

### 10.1 Token Budget 续写机制

当模型在 budget 未用完时就停下了，注入 meta 消息促使模型继续：

```typescript
if (decision.action === 'continue') {
  messages.push(createUserMessage({ content: decision.nudgeMessage, isMeta: true }))
  continue  // 回到循环顶部
}
```

当检测到"边际递减"（diminishing returns）时提前停止。

**Harness 意义**：控制模型的输出量——既不让它生成太少（续写），也不让它生成无意义内容（边际递减检测）。

### 10.2 Context Efficiency 提醒注入

当历史 token 增长了一定量但模型没有主动使用 SnipTool 清理时，注入提醒消息。

**Harness 意义**：不是强制执行压缩，而是"提醒"模型它有清理上下文的能力。把决策权留给模型，但确保它不会忘记这个能力。

### 10.3 记忆提取 Agent 的 Prompt 约束

```
"You have a limited turn budget.
 FileEdit requires a prior FileRead of the same file,
 so the efficient strategy is:
   turn 1 — issue all FileRead calls in parallel
   turn 2 — issue all FileWrite/FileEdit calls in parallel
 Do not interleave reads and writes across multiple turns."
```

**Harness 意义**：通过 prompt 告诉子 Agent **最优执行策略**，而不是靠它自己摸索。代码不强制（`maxTurns: 5` 足够灵活），但 prompt 引导。

### 10.4 Compact 摘要的结构化 Prompt

Full Compact 要求模型先在 `<analysis>` 标签中分析（草稿），然后在 `<summary>` 标签中输出 9 章节结构化摘要。`<analysis>` 在后处理时被剥离。

**Harness 意义**：强制模型"先想后写"——analysis 标签相当于 scratchpad，让模型在输出最终摘要前有一个组织思路的空间。

### 10.5 附件系统的精确注入时机

不同的上下文信息在不同时机注入：

```
循环开始 → System prompt（静态）+ User context（MEMORY.md 索引）
API 调用前 → 压缩流水线处理后的消息
工具执行后 → 排队命令 + 记忆预取结果 + 技能发现结果
MCP 工具刷新 → 工具池更新
```

**Harness 意义**：信息注入不是"全部塞进去"，而是在最合适的时机注入最相关的信息。

---

## 11. 总结：设计原则提炼

### 原则 1：渐进式降级（Graceful Degradation）
上下文压缩的五层流水线、max_output_tokens 的三级恢复、413 的三级恢复链——都遵循"先用最轻量的手段，不行再升级"的模式。

### 原则 2：Fail-Closed 默认值
工具并发安全性默认 false、权限默认保守、新工具不设置就走最严格规则。安全约束的默认方向是"收紧"。

### 原则 3：显式状态追踪
transition.reason 记录恢复原因、权限来源追踪到 OTel、lastSummarizedMessageId 追踪压缩游标——关键决策点都有可审计的追踪。

### 原则 4：成本前移（Amortize Cost）
SM-Compact 把摘要生成前移到后台；记忆预取前移到工具执行期间；Forked Agent 共享 prompt cache。贵的操作尽可能提前做或并行做。

### 原则 5：索引-内容分离
MEMORY.md 是索引；ToolSearch 的名字列表是索引；deferred tools 的 delta 消息是索引。轻量索引始终可见，重量级内容按需加载。

### 原则 6：精确隔离
siblingAbortController 只杀兄弟不杀主循环；子 Agent 工具集收窄；沙箱执行隔离。错误和权限的影响范围被精确限定。

### 原则 7：利用自然等待时间
流式工具执行利用模型输出时间；异步预取利用工具执行时间；post-sampling hooks 利用用户阅读时间。不是"减少等待"而是"在等待中做有用的事"。

### 原则 8：双重验证（Belt and Suspenders）
FileEditTool 在 validate 和 call 两个阶段都做新鲜度检查；Windows 下时间戳 + 内容对比双保险；readFileState LRU + loadedNestedMemoryPaths 永久 Set 双层去重。关键路径上有冗余安全措施。

### 原则 9：模型自主性 + 系统兜底
SnipTool 让模型自己决定删什么，系统提供提醒兜底；Token Budget 让模型自己决定什么时候停，系统提供续写和强制停止兜底。给模型决策权但不完全信任。

### 原则 10：缓存感知设计
所有修改消息内容的操作都需要考虑对 prompt cache 的影响。热缓存时用 cache_edits 不改内容；工具排序保证 cache key 稳定；Forked Agent 共享 cache key。
