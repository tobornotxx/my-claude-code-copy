# 上下文管理策略笔记
## Tool Result Budget：
每个工具都有 `maxResultSizeChars` 属性：
- 大多数工具：有限值（超出存磁盘）
- `FileRead`：`Infinity`（不持久化，避免 Read→file→Read 循环）
进入Query Loop后立刻进入此步。对每个工具调用结果进行一次处理，如果超出字符最大上限则存为文档，context中只保留工具名和文件路径引用。

**触发条件**：
- **单工具上限**：每个工具结果超过 `maxResultSizeChars`（系统级硬顶 **50,000 字符**，定义在 `toolLimits.ts`）→ 持久化到磁盘，context中替换为 ~2KB 预览 + 文件路径
- **单消息聚合上限**：同一条 user message 内所有 tool_result 块合计超过 **200,000 字符** → 从最大的块开始逐个持久化，直到总量降到预算内
- **执行时机**：Pipeline 第 1 步，`enforceToolResultBudget()` 在每轮 query loop 开头执行
- 远程配置覆盖：`tengu_satin_quoll`（单工具）、`tengu_hawthorn_window`（单消息）

## History Snip:

- SnipTool为一个工具，模型可调用，使用消息id删除对应的消息内容；每增长一定长度的token(~10k tokens)而模型没有调用snip tool,会有注入的context_efficiency附件信息提醒使用SnipTool.
- 每轮query loop开头，会调用一次snipCompactIfNeeded(), 读取之前SnipTool的标记，并会移除对应的信息

**触发条件**：
- **snipCompactIfNeeded()**：Pipeline 第 2 步，每轮 query loop 开头无条件执行，读取之前 SnipTool 写入的标记并移除对应内容
- **context_efficiency 提醒注入**：自上次 snip/nudge/compact boundary 以来，token 增长超过 **~10,000 tokens** → 注入 `context_efficiency` 附件提醒模型使用 SnipTool
- **前提门控**：`feature('HISTORY_SNIP')` 开启 且 `isSnipRuntimeEnabled()` 为 true
- **重置时机**：模型调用 SnipTool / 出现 snip boundary / compact boundary / 新 nudge 时，增长计数器重置


## Microcompact:
Microcompact 有**两条路径**，根据场景自动选择。**执行时机**：Pipeline 第 3 步，在 API 调用之前执行（`microcompactMessages()`）。

#### 路径 A：Cached Microcompact（主路径，热缓存时）

**触发条件**：API 上一次响应带回了 `cache_creation_input_tokens`（表明服务端有热缓存命中）。`feature('CACHED_MICROCOMPACT')` 开启。

**机制**：利用 Claude API 的 `cache_edits` 能力，在保持其余前缀Prompt token KV cache重利用的前提下定向删除部分tokens的cache，避免更改上下文后api对于整个之前已计算的context cache必须重新计算一次kv的问题。

**关键**：消息内容完全不变（保护 prompt cache），只在 API 请求中附带 `cache_edits` 指令。API 响应会带回 `cache_deleted_input_tokens` 报告实际删除了多少 token。

#### 路径 B：Time-Based Microcompact（冷缓存时）

**触发条件**：`(当前时间 − 上一条 assistant 消息的时间戳) > 60 分钟`。60 分钟对应服务端 1 小时 cache TTL，保证缓存必然已过期。仅主线程生效（子 Agent 生命周期太短不适用）。远程配置：`tengu_slate_heron`（含 enabled 开关、gapThresholdMinutes、keepRecent）。

**行为**：因为api的kv cache已经没有了，所以可以随便删之前的context结果精简上下文。保留最近 N 个（默认 5）可压缩的工具结果，其余直接**清空内容**为 `'[Old tool result content cleared]'`。这条路径是**真正修改消息内容**的。

删除的是旧的tool calls, 且删除的tool call限定在一个名单里：FileRead, Shell, Grep, Glob, WebSearch, WebFetch, FileEdit, FileWrite
因为此类tool call时效短！到达上限后直接保存最近的几次tool call，剩下的直接删除。

## Context Collapse
把旧的对话轮次”归档”成摘要，维护一个类似 git log 的结构，每一轮做了什么事、结论是什么都保留为独立的摘要记录，不是混成一坨。
**触发条件**：
- **前提门控**：`feature('CONTEXT_COLLAPSE')` 开启 且 `isContextCollapseEnabled()` 为 true
- **执行时机**：Pipeline 第 4 步，在 autocompact **之前**执行（`applyCollapsesIfNeeded()`）
- **与 autocompact 的关系**：如果 collapse 将 token 数降到了 autocompact 阈值以下，autocompact 就变成 no-op（保留粒度更细的结构化摘要，不会被压成一坨）
- **413 恢复链中也会触发**：收到 413 错误时，三级恢复链的第 1 级是 `recoverFromOverflow()` 提交暂存的 collapses → 第 2 级才是 reactive compact
- collapsed view 是**读时投影**（read-time projection），不修改原始消息数组，摘要存在 collapse store 里
## Autocompact
使用LLM压缩上下文为一段话。

**触发条件**：
- **阈值公式**：`getEffectiveContextWindowSize(model) − 13,000 tokens`（AUTOCOMPACT_BUFFER_TOKENS = 13,000）
- **触发判断**：`isAutoCompactEnabled() && tokenUsage >= autoCompactThreshold`（token 计数会减去 snipTokensFreed 避免 snip 后误触发）
- **执行时机**：Pipeline 第 5 步，在 context collapse 之后执行
- **被抑制的情况**：
  - Context Collapse 已启用且 `isContextCollapseEnabled()` + `isAutoCompactEnabled()` 同时为 true → collapse 接管，autocompact 不做预检
  - Reactive Compact 实验（`tengu_cobalt_raccoon`）启用时 → autocompact 的主动触发被抑制
- **熔断器**：连续失败 **3 次**后停止重试（MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3），防止 context 不可恢复时浪费 API 调用
- **环境变量覆盖**：`CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` 可设百分比（如 `50` = 50% 时触发），取百分比阈值和默认阈值的较小值
- **额外**：context window ≥ 1M 时，token 用量超过 effective window 的 **25%** 会注入 `compaction_reminder` 附件提醒模型

## Reactive Compact
与 autocompact 的根本区别：autocompact 是**主动预防**（在 token 快满时提前压缩），reactive compact 是**被动响应**（等 API 真的返回 413 错误再压缩）。

当 reactive compact 启用时，它会**抑制** autocompact 的主动触发，让 API 错误自然发生，然后在 query.ts 的错误恢复路径中处理.

**触发条件**：
- **触发错误类型**：
  - Prompt too long（HTTP **413**，`isWithheld413`）
  - Media size exceeded（`isWithheldMedia`）
  - Max output tokens exceeded（`isWithheldMaxOutputTokens()`）
- **错误扣留机制**：错误在流式过程中检测到，但**不立即抛出**，先尝试恢复；恢复失败才向上暴露
- **防循环**：每次 query 只尝试一次（`hasAttemptedReactiveCompact` 标志），恢复后设置 `transition.reason = 'reactive_compact_retry'` 防止下一轮重复进入
- **前提门控**：`feature('REACTIVE_COMPACT')`
- **恢复链优先级**：
  1. Context Collapse drain（如果可用，先提交暂存的 collapses）
  2. Reactive compact（压缩对话重试）
  3. PTL 特殊处理（如果是 compact agent 自身的请求收到 413：截断头部消息，最多重试 3 次）

---

# 工具系统设计笔记

## 核心理念：零继承，纯工厂函数——buildTool到底是什么

40多个工具没有用任何类继承体系，全部通过一个 `buildTool()` 工厂函数创建。

### 先说清楚：这不是经典的"工厂模式"

教科书上的工厂模式是这样的：你有一个基类 Animal，子类有 Dog/Cat/Bird，工厂函数根据参数决定创建哪个子类：
```
function createAnimal(type) {
  if (type === 'dog') return new Dog()
  if (type === 'cat') return new Cat()
}
```
重点是**隐藏具体类型**，调用方不需要知道创建的是哪个子类。

`buildTool()` **不是这个东西**。它不根据参数选择不同的类，它做的事情更简单——接收一个配置对象，**合并上默认值**，返回一个完整的工具对象。更准确的叫法是**"带默认值的对象工厂"**，或者叫**Builder模式的简化版**。

### buildTool实际做了什么

源码其实非常短（去掉类型定义只有5行）：

```typescript
const TOOL_DEFAULTS = {
  isEnabled: () => true,              // 默认启用
  isConcurrencySafe: () => false,     // 默认不允许并发（安全保守）
  isReadOnly: () => false,            // 默认认为会写（安全保守）
  isDestructive: () => false,         // 默认非破坏性
  checkPermissions: (input) =>        // 默认放行（交给通用权限系统）
    Promise.resolve({ behavior: 'allow', updatedInput: input }),
  toAutoClassifierInput: () => '',    // 默认不参与安全分类
}

function buildTool(def) {
  return {
    ...TOOL_DEFAULTS,        // 先铺默认值
    userFacingName: () => def.name,  // 默认用工具名
    ...def,                  // 用户定义覆盖默认值
  }
}
```

就是一个**对象展开合并**：默认值在前，用户定义在后，同名属性后者覆盖前者。

### 这解决了什么实际问题

**问题场景**：Tool 接口有大约30多个方法/属性（call, description, validateInput, checkPermissions, isConcurrencySafe, isReadOnly, renderToolUseMessage, renderToolResultMessage, mapToolResultToToolResultBlockParam, getToolUseSummary, getActivityDescription, ...）。

如果用**继承**：
```typescript
class BaseTool {
  isEnabled() { return true }
  isConcurrencySafe() { return false }
  isReadOnly() { return false }
  // ... 30 个方法都要在基类里写默认实现
}

class GlobTool extends BaseTool {
  isConcurrencySafe() { return true }  // 覆盖
  isReadOnly() { return true }         // 覆盖
  call() { /* ... */ }
  // ...
}
```

继承的问题：
1. **基类膨胀**：30个方法的默认实现全堆在一个类里，随着新工具加入还会不断加新方法
2. **"上帝类"反模式**：BaseTool 什么都知道一点，但什么都不精通
3. **覆盖不明显**：子类里哪些方法是覆盖的、哪些是继承的，看代码不直观
4. **this绑定问题**：继承链里的 this 指向容易出bug，特别是在异步回调里

如果用 **buildTool + 配置对象**：
```typescript
// GlobTool — 简单工具，只声明自己需要的
export const GlobTool = buildTool({
  name: 'Glob',
  isConcurrencySafe() { return true },   // 我是只读的，可并发
  isReadOnly() { return true },
  call(input, context) { /* ... */ },
  // isEnabled, checkPermissions 等全用默认值，不需要写
})

// BashTool — 复杂工具，声明了大量自定义行为
export const BashTool = buildTool({
  name: 'Bash',
  isConcurrencySafe(input) {             // 根据输入动态判断
    return isReadOnlyCommand(input.command)
  },
  isReadOnly(input) { return isReadOnlyCommand(input.command) },
  isDestructive(input) { return isDestructiveCommand(input.command) },
  checkPermissions(input, ctx) { /* 复杂的安全检查 */ },
  validateInput(input, ctx) { /* 9道校验 */ },
  call(input, ctx) { /* 沙箱执行、输出处理、自动后台化... */ },
  // ... 还有十几个自定义方法
})
```

**每个工具一眼就能看到它自定义了什么，没自定义的全用默认值**。不需要去翻基类找"我到底继承了什么行为"。

### 默认值的设计哲学："安全保守"

注意默认值的选择不是随便定的：
- `isConcurrencySafe` 默认 **false**（不允许并发）：新工具如果忘了设置，不会意外被并发执行导致竞态
- `isReadOnly` 默认 **false**（假设会写）：新工具如果忘了设置，权限系统会保守地当它是写操作
- `checkPermissions` 默认 **allow**（放行）：因为有通用权限系统兜底，工具级权限是可选的附加层

这叫 **fail-closed**——默认选最保守的选项，开发者必须**主动声明**才能放宽。源码注释里也明确说了：`Defaults (fail-closed where it matters)`。

### TypeScript类型体操：为什么不是简单的 Partial

`buildTool` 的类型签名看起来简单，实际有一层精巧的设计。它用了一个 `BuiltTool<D>` 类型，作用是：

- **输入侧**（ToolDef类型）：那7个有默认值的方法是 **optional**，你可以不写
- **输出侧**（BuiltTool类型）：所有方法都是 **required**，TypeScript保证你拿到的对象一定有所有方法

这意味着**使用工具的代码不需要做任何空值检查**。不用写 `tool.isReadOnly?.() ?? false`，直接 `tool.isReadOnly(input)` 就行，因为类型保证它一定存在。

如果用简单的 `Partial<Tool>` + 运行时默认值，每个使用点都得处理 undefined 的情况。`buildTool` 把这个问题在**类型层面**一次性解决了。

### 为什么不用接口+默认实现（类似Java的default method）

TypeScript也可以这样做：
```typescript
interface Tool {
  isReadOnly(): boolean  // 没有默认值，必须实现
}
```
但问题是**全有或全无**——要么每个工具都必须实现30个方法（太啰嗦），要么全部optional然后每个调用点都做undefined检查（太麻烦）。

`buildTool` 的方案是第三条路：**定义侧optional，使用侧required**，中间用一个运行时展开操作桥接。这是TypeScript特有的优势，Java/Python做不到这么优雅。

### 总结

`buildTool` 不是经典工厂模式，它是一个**配置合并+类型补全**的工具：
1. 运行时：`{...默认值, ...用户定义}` — 对象展开合并，后者覆盖前者
2. 类型层面：输入optional，输出required — 消除所有下游的空值检查
3. 默认值设计：安全保守（fail-closed），新工具不设置就走最严格的规则
4. 最终效果：简单工具写20行，复杂工具写1000行，接口形状完全一样，Agent Loop不关心内部差异

## 动态工具池：运行时组装，不是编译时写死

工具列表不是一个静态数组，而是每个session启动时根据feature flag、用户权限、环境配置**动态拼装**的。比如：
- `SleepTool` 只在 PROACTIVE/KAIROS 模式下才加载
- `REPLTool` 只对内部用户可用
- MCP（外部插件）工具在运行时合并进来

最后所有工具**按名称排序**再交给API。这个排序不是为了好看，是为了保证prompt cache的key稳定——如果工具顺序每次不一样，API会认为是不同的prompt，cache就废了。

## Deferred Loading：不全部告诉模型

启用ToolSearch后，部分工具标记为 `shouldDefer: true`，它们的schema**不发给模型**。模型只有在通过 ToolSearch 工具主动查找后才能使用它们。

**为什么**：40多个工具的schema本身就要吃很多token。模型大部分时候只用到 Bash/FileRead/FileEdit 等核心工具，其余的schema白白占着prompt空间。延迟加载把工具池分成"常驻"和"按需"两层，节省prompt token。

### 模型怎么知道有哪些隐藏工具？

模型**不是完全靠猜**。系统通过 `deferred_tools_delta` 附件消息，在 `<system-reminder>` 中告诉模型**所有被隐藏的工具的名字列表**。每当工具池变化（比如新的MCP服务器连上来），会计算一个增量delta（哪些新增、哪些移除），追加通知。所以模型看到的是类似这样的信息："以下工具可通过ToolSearch获取：mcp__slack__send_message, mcp__github__create_issue, NotebookEditTool, ..."。

**模型知道名字，但不知道参数schema**——所以它知道"有一个叫slack的工具"，但不知道怎么调用。必须先调用 `ToolSearch` 查一下，拿到完整的JSONSchema定义后才能调用。

### ToolSearch的搜索机制：关键词打分，不是向量搜索

ToolSearch**不用向量/embedding**，是一个纯关键词匹配+加权打分系统：

**两种查询模式**：
1. **精确选择**：`select:Read,Edit,Grep` — 按名字直接查找（支持逗号分隔多选）
2. **关键词搜索**：`notebook jupyter` — 对所有deferred工具做关键词打分排序

**关键词打分规则**（每个搜索词独立计分，累加）：
- 工具名的拆词**完全匹配**（如搜"slack"命中了 mcp__slack__send 的 "slack" 词段）：MCP工具+12分，普通工具+10分
- 工具名的拆词**部分包含**（如搜"note"命中了"notebook"）：MCP+6分，普通+5分
- 工具的 `searchHint` 字段匹配（人工策划的搜索关键词）：+4分
- 工具的完整description（即prompt()返回的内容）中**词边界匹配**：+2分

**工具名怎么拆词**：
- MCP工具按下划线拆：`mcp__slack__send_message` → ["slack", "send", "message"]
- 普通工具按CamelCase拆：`NotebookEditTool` → ["notebook", "edit", "tool"]

**还支持必需词过滤**：用 `+slack send` 语法，`+slack` 是必需的（工具名或描述必须包含），其余词只用于排序。先过滤再打分，避免大量无关工具参与评分。

### ToolSearch返回结果怎么变成可调用的工具？

搜索匹配后，ToolSearch工具返回的不是普通文本，而是 `tool_reference` 类型的内容块（API层面的特殊类型）。API收到 tool_reference 后，会把对应工具的完整JSONSchema展开给模型看——等价于该工具从一开始就在prompt的tools列表里。之后模型就可以正常调用了。

所以整个流程是：**名字列表告知 → 模型判断需要哪个 → 调ToolSearch查 → API展开schema → 正常调用**。不是盲猜，是有索引的按需加载。

## 流式工具并行：模型还在说话，工具已经在跑

普通Agent的流程是：等模型输出完 → 解析工具调用 → 执行 → 返回结果。Claude Code不等——模型流式吐出一个tool_use block，`StreamingToolExecutor` 立刻把这个工具加入执行队列。模型继续生成后面的内容时，前面的工具已经在运行了。

**并发规则很简单**：
- 只读工具（grep, 读文件等，`isConcurrencySafe=true`）：可以互相并行
- 写操作工具（bash写命令, 编辑文件等，`isConcurrencySafe=false`）：必须独占，前面的全跑完才能开始

**错误级联设计**：Bash命令出错时，会取消所有正在并行的兄弟工具（因为bash命令之间往往有隐式依赖，比如前一条创建目录，后一条写文件）。但其他工具（如FileRead）出错不会级联，因为它们是独立的。这里用了一个单独的 `siblingAbortController`，只杀兄弟不杀主循环，隔离得很干净。

## 工具执行的完整链路：6层检查

一个工具从模型输出到实际执行，要过6道关：
1. **查找工具定义**（支持别名回退，找不到直接返回错误）
2. **Zod schema校验**（类型检查，如果是deferred工具还没加载schema，提示先用ToolSearch）
3. **自定义业务校验**（`validateInput`，比如FileEdit检查文件是否存在）
4. **Pre-Hook**（外部钩子，可以拦截或修改输入）
5. **权限决策**（hook已决定→用hook的；否则→弹确认框问用户）
6. **实际执行**（带OpenTelemetry链路追踪）

**设计亮点**：权限来源被完整追踪（hook批准/用户临时允许/永久规则/交互确认/默认配置），方便事后审计"这个操作是谁放行的"。

## BashTool为什么这么复杂（18个文件）

BashTool不只是 `exec(command)`，它要处理：
- **安全**：解析命令结构分类危险程度，macOS用sandbox-exec沙箱，Linux用seccomp
- **读写分类**：`grep/find/cat` 是只读可并发，`git push/rm` 是写操作要独占
- **sed特殊处理**：检测到 `sed -i` 时UI变成文件编辑样式，解析表达式显示diff预览
- **大输出处理**：超过30K字符的输出持久化到磁盘（和Tool Result Budget的思路一样）
- **自动后台化**：助手模式下命令跑超过15秒自动转后台，不阻塞交互

**启发**：一个"简单"的bash工具，实际上藏着安全沙箱、并发控制、UI适配、资源管理等大量关注点。但从外部看它仍然只是一个 `buildTool({...})` 对象，所有复杂性被封装在内部，不会泄漏到Agent Loop里。这就是工厂函数模式的好处——接口统一，内部复杂度自由。

## FileEditTool的原子性设计

编辑文件不是简单的"读-改-写"，而是有**新鲜度检查**防止竞态：
- validate阶段检查一次文件修改时间
- call阶段执行前**再检查一次**（防止validate和call之间文件被外部改了）
- Windows下时间戳精度有问题，所以时间戳对比失败后还会做**内容对比**兜底

写入后还要通知LSP服务器和VS Code编辑器同步状态，更新readFileState的缓存。整个过程保持原始文件的编码和换行符格式不变。

## 总结：可以学到的设计模式

1. **工厂函数 > 继承**：当"子类"之间差异大于共性时，用配置对象+默认值比继承树更灵活
2. **动态组装+排序稳定性**：运行时拼装能力池，但要注意序列化顺序对缓存的影响
3. **只读/写操作分类**：用一个简单的布尔标记就实现了高效的并发控制策略
4. **流式提前执行**：不等上游全部完成再开始下游，能并行就并行，延迟最小化
5. **错误隔离**：siblingAbortController这种设计——出错影响范围可控，不会一个错误炸掉整个系统
6. **接口统一，内部自由**：不管工具内部多复杂（BashTool 18个文件），对外都是同一个接口形状

---

# 记忆系统笔记

## MEMORY.md 和 CLAUDE.md 不是一回事

这两个容易混淆：
- **CLAUDE.md**：人写的项目规则，放在项目目录里，随git提交，启动时加载一次，进system prompt。相当于"项目README给AI看的版本"
- **MEMORY.md**：AI自动提取+用户手动编辑的跨会话知识，放在 `~/.claude/projects/<项目hash>/memory/` 下，每次query重新加载

一个是**指令**（你要遵守的规矩），一个是**记忆**（上次学到的东西）。

## 记忆文件怎么组织的

```
~/.claude/projects/<项目hash>/memory/
  ├─ MEMORY.md           ← 这是索引！不是记忆本身
  ├─ user_role.md         ← 具体的记忆文件
  ├─ feedback_testing.md
  ├─ project_context.md
  └─ team/               ← 团队共享记忆（TEAMMEM feature）
       ├─ MEMORY.md
       └─ shared_conventions.md
```

**MEMORY.md 是索引文件**，每一行就是一个指向具体记忆文件的指针+一句话摘要：
```markdown
- [User Testing Preferences](feedback_testing.md) — Jest over Vitest, docker-compose
- [Deployment Pipeline](project_deploy.md) — Uses GitHub Actions + ArgoCD
```
索引有硬限：200行 + 25KB。超了会截断并附加警告。

每个记忆文件是带 frontmatter 的 Markdown：
```markdown
---
name: User Testing Preferences
description: How the user likes tests written and run
type: feedback
---
具体内容...
```

**四种记忆类型**（封闭分类，不能随便加）：
- `user`：用户是谁、怎么协作
- `feedback`：用户的纠正和偏好（"我不喜欢你这样做"）
- `project`：项目上下文（不能从代码推导出来的信息）
- `reference`：工具/API参考文档

**明确不该存的**：能从代码直接看出来的东西（架构风格、命名约定）、Git历史、CLAUDE.md里已有的规则、大段代码。

## 记忆怎么生成的：后台子Agent自动提取

每次模型回复完毕（Agent Loop走到stop hooks阶段），系统会 fire-and-forget 启动一个**后台子Agent**来分析刚才的对话，提取值得记住的东西。

**执行流程**：
1. 主Agent回复完毕 → stop hooks触发 `executeExtractMemories()`
2. 用 `runForkedAgent()` fork主对话（共享prompt cache，不额外花钱计算前缀KV）
3. 子Agent拿到受限的工具权限：只能读任意文件（FileRead/Grep/Glob/只读Bash），只能往 memory/ 目录写文件
4. 子Agent的策略是**两轮搞定**：第1轮并行读取需要的文件，第2轮并行写入所有记忆文件。这是 **prompt 级约束而非代码强制**，因为 FileEdit 要求目标文件已经被 FileRead 过（新鲜度校验），如果交叉读写（读A→写A→读B→写B），4个文件就要 8 轮，超出 `maxTurns: 5` 限制会被截断。批量读→批量写只要 2 轮
5. 写完后在REPL里显示 "Memory saved: xxx.md"

**关键的安全设计**：
- **互斥**：如果主Agent自己已经在这轮写了记忆（用户说"记住这个"），后台提取就跳过，不重复
- **节流**：不是每轮都提取，有计数器控制频率。具体：闭包里一个 `turnsSinceLastExtraction` 计数器，每次 eligible turn 自增，达到阈值（远程配置 `tengu_bramble_lintel`，默认 1）后才执行提取并重置为 0。默认 1 = 每个 eligible turn 都提取，如果配置改为 3 就是每 3 轮提取一次。“eligible turn”本身已经过了前置门控（只对主 Agent、auto memory 启用、非远程模式等）
- **合并**：提取正在进行时又来了新一轮，不会启第二个并发提取，而是暂存上下文等当前完成后做"尾随运行"
- **优雅关闭**：用户退出时最多等60秒让正在进行的提取完成（`.unref()` 保证不阻塞进程退出）

## 记忆怎么召回的：三条路径

记忆不是一股脑全塞进上下文，而是通过三条独立路径按需注入：

### 路径A：静态注入（说明 + 索引，每次都有）

路径 A 通过两个机制注入内容：

**机制 1：System Prompt — 记忆系统使用说明**

`loadMemoryPrompt()` 调用 `buildMemoryLines()`，注入纯指令文本（**不含 MEMORY.md 内容**）：
- 记忆系统说明（告诉模型“你有一个持久化文件系统”）
- 四种类型定义 + 什么该存什么不该存
- 如何保存记忆的两步流程（写文件 + 更新 MEMORY.md 索引）

**机制 2：User Context — MEMORY.md 索引内容**

`getUserContext()` → `getMemoryFiles()` → `getClaudeMds()` 路径中，当 auto memory 启用时会读取 MEMORY.md 文件内容，标记为 `'AutoMem'` 类型注入 user context。受 `tengu_moth_copse` flag 控制（默认加载）。加载时受 200行/25KB 截断限制。

所以模型看到的是：system prompt 中的使用说明 + user context 中的 MEMORY.md 索引（标题列表）。但看不到具体记忆文件的内容——那些由 Path C 或模型主动 FileRead 提供。

### 路径B：CLAUDE.md 项目指令延迟加载（工具触发）

> 注意：这与 auto-memory（MEMORY.md）系统**无关**。加载的是子目录的 **CLAUDE.md 项目指令**，不是记忆文件。代码里叫“嵌套记忆”是因为它和记忆系统共享同一个模块（`claudemd.ts`）。

当模型用 FileReadTool 读了一个文件，系统沿着该文件路径的目录树向上查找并加载各级 CLAUDE.md。如读了 `/src/auth/handler.ts`，就加载 `/src/auth/CLAUDE.md`、`/src/CLAUDE.md`（如果存在）。目的是子目录可以有自己的规则（如 `tests/CLAUDE.md` 说“用 pytest”），避免启动时一口气加载所有子目录规则浪费 token。

**去重用两层**：
- `loadedNestedMemoryPaths`（永久Set，不驱逐）——防止同一个CLAUDE.md被反复注入
- `readFileState`（100项LRU缓存，会驱逐）——兜底层

为什么需要两层？`readFileState` 是 LRU，忙碌会话里旧条目会被驱逐掉，导致同一个CLAUDE.md被驱逐后又被当成"新的"重复注入。加一个永久Set兜底就解决了。

### 路径C：相关记忆召回（最精密的，用Sonnet做语义选择）

Path C **不依赖 MEMORY.md 索引**。它直接扫描 memory/ 目录下所有 .md 文件的 frontmatter（显式排除 MEMORY.md 本身），做语义匹配。

每次用户输入时，系统**异步预取**可能相关的记忆：

1. **扫描** memory/ 目录，每个 .md 文件读前30行提取 frontmatter（name, description, type），按修改时间排序，最多200个文件
2. **构建manifest**：把所有记忆的元信息格式化成一个列表，类似：`- [feedback] deploy_issues.md (2026-04-10): staging部署常见问题`
3. **调Sonnet做语义选择**：把manifest和用户输入给Sonnet，让它选最多5个最相关的记忆
4. **读取选中文件**：每文件最多200行 + 4KB（超出被截断，模型看不到后面的内容；但模型可以自己 FileRead 读完整内容）
5. **去重**：排除已被其他路径加载过的文件
6. **附加新鲜度警告**：超过1天的记忆会附带提醒"这条记忆可能过时了，引用前先验证"

**预取时机很巧妙**：用户输入后立刻异步启动，利用工具执行的5-30秒时间并行完成。工具执行完后直接消费结果，不额外等待。

**会话级节流**：累计召回的记忆字节不超过 60*1024 = 61,440 字节。compact压缩上下文后旧附件消失，计数器自然重置，又可以召回新记忆了。

### 三条路径的关系

| 路径 | 注入什么 | 模型看到什么 |
|---|---|---|
| **Path A** | 使用说明 + MEMORY.md 索引 | “你有这些记忆”（标题列表，不含内容） |
| **Path B** | 子目录的 CLAUDE.md | 项目指令（与记忆无关） |
| **Path C** | 具体记忆文件的实际内容 | 相关记忆的具体知识 |

Path A 给模型一个目录（“你有这些记忆”），Path C 在模型还没来得及主动查之前就把可能需要的具体内容预取好了。模型也可以看着 Path A 的索引自己 FileRead 具体文件——但 Path C 更快（异步预取，零延迟）。

## 记忆和上下文压缩（Compact）怎么配合

compact对记忆系统有影响：
- `readFileState` 被清空 → 之前读过的记忆文件缓存失效
- `loadedNestedMemoryPaths` 被清空 → 嵌套CLAUDE.md可以重新注入
- 旧的记忆附件从历史中消失 → 61,440 字节召回限额自然重置
- 但 MEMORY.md 索引在 system prompt 里 → 不受compact影响，每次都重新加载

相当于 compact 后记忆系统"重启"了一轮：索引还在，但具体内容需要重新按需召回。

## 设计启发

1. **索引和内容分离**：MEMORY.md是索引（轻量，始终可见），具体记忆文件是内容（重量，按需加载）。和数据库的索引思路一样——不需要把全部数据塞进内存，只需要一个足够小的索引来决定"需要加载哪些"
2. **三层召回粒度**：静态全量索引（粗）→ 路径触发的目录级加载（中）→ LLM语义选择的文件级加载（细）。越精细的路径成本越高（要调Sonnet），但精度也越高
3. **异步预取+并行利用等待时间**：不是"需要记忆→停下来去取→拿到后继续"，而是"入口处异步发射→利用工具执行的等待时间→完成后直接消费"，用户感知延迟为零
4. **写入端的互斥和合并**：主Agent和提取Agent不会同时写记忆（互斥），多轮提取请求不会并发（合并），用闭包维护状态而不是全局变量
5. **新鲜度管理**：记忆不是写了就永久可信，超过1天就提醒验证。承认"记忆会过时"这个现实，而不是假装它永远正确

---

# SubAgent 系统笔记

## 两种"轻重"机制：sideQuery vs runForkedAgent

系统有两种完全不同的"子调用"方式，解决不同的问题：

### sideQuery — 一次性轻量调用

`sideQuery()` 就是一个带了标准化处理的 `messages.create()` API 调用封装。**不走 query loop**，不执行工具，不追踪 usage，不记录 transcript。

**它负责**：
- 加标准的 CLI attribution header（OAuth 验证需要）
- 处理 model string 标准化（去掉 `[1m]` 后缀）
- 加正确的 beta flags
- 计算 fingerprint 给 OAuth 验证
- 可选的 system prompt 前缀、structured output（JSON schema）、thinking 配置

**使用场景**：全是"问模型一个问题，拿答案"的场景——
- 记忆召回的 Sonnet 语义选择（`selectRelevantMemories`）
- 权限决策分类（`yoloClassifier`）
- 模型能力验证
- Session 搜索

**关键限制**：默认 `max_tokens: 1024`，没有工具、没有多轮、没有 prompt cache 协调。

### runForkedAgent — 完整 Agent 循环

`runForkedAgent()` 启动一个**完整的 query loop**（就是主循环用的同一个 `query()` 函数），带工具执行、多轮对话、usage 累计、transcript 记录。

**核心卖点是 prompt cache 共享**：fork 出来的子 Agent 和父 Agent 用完全相同的 system prompt + tools + 历史消息前缀调 API，所以 Anthropic API 服务端的 KV cache 可以直接复用。效果是子 Agent 的第一次 API 调用~90% 的 input token 都是 cache_read（便宜 10 倍）。

**流程**：
1. 从父 Agent 拿 `CacheSafeParams`（system prompt、userContext、systemContext、toolUseContext、历史消息）
2. 用 `createSubagentContext()` 创建隔离的上下文
3. 拼接 `[...forkContextMessages, ...promptMessages]` 作为初始消息
4. 调 `query()` 进入完整 Agent 循环
5. 循环结束后返回 `{ messages, totalUsage }`
6. 可选记录 sidechain transcript（用于 session resume）

## CacheSafeParams：缓存共享的核心

子 Agent 要共享父 Agent 的 prompt cache，**必须保证 API 请求的前缀字节完全一致**。`CacheSafeParams` 封装了所有影响缓存键的参数：

```typescript
type CacheSafeParams = {
  systemPrompt: SystemPrompt      // 必须和父一模一样
  userContext: { [k: string]: string }  // 必须一样
  systemContext: { [k: string]: string } // 必须一样
  toolUseContext: ToolUseContext    // 包含 tools、model、thinkingConfig
  forkContextMessages: Message[]   // 父的历史消息（cache 前缀）
}
```

**陷阱**：如果子 Agent 设了 `maxOutputTokens`，会影响 `budget_tokens`（thinking 配置的一部分），而 thinking 配置是缓存键的一部分 → **缓存失效**。所以只有不需要缓存共享的场景（如 compact summary）才设 `maxOutputTokens`。

**写入时机**：`handleStopHooks()` 每轮结束时调 `saveCacheSafeParams(params)` 存到模块级全局变量，post-turn hook（记忆提取、/btw 等）直接 `getLastCacheSafeParams()` 取用。

## createSubagentContext：状态隔离的关键

子 Agent 不能随便读写父 Agent 的状态。`createSubagentContext()` 做了精确的隔离设计：

**默认隔离（clone or no-op）**：
| 状态 | 隔离方式 | 原因 |
|---|---|---|
| `readFileState` | **clone**（不是新建） | clone 保证子 Agent 对父消息中的 tool_use_id 做相同的替换决策 → 保持 cache 前缀一致 |
| `contentReplacementState` | **clone** | 同上，避免 budget 决策分歧导致 wire 前缀不同 |
| `setAppState` | **no-op** | 防止子 Agent 修改父的 UI 状态 |
| `setInProgressToolUseIDs` | **no-op** | 隔离进度指示 |
| `setResponseLength` | **no-op** | 隔离 metrics |
| `abortController` | **新建 child controller** | 父 abort → 子也 abort（单向传播），子 abort 不影响父 |
| `nestedMemoryAttachmentTriggers` | **新建空 Set** | 每个子 Agent 独立追踪 |
| `toolDecisions` | **undefined** | 全新的 |

**特殊处理**：
- `setAppStateForTasks`：即使 `setAppState` 是 no-op，**任务注册/kill 必须穿透到根 store**。否则子 Agent 启动的后台 bash 任务无法被 kill（变成僵尸进程）
- `localDenialTracking`：如果 setAppState 是 no-op，denial 计数器也需要本地化，否则重试间的 denial 累积丢失
- `shouldAvoidPermissionPrompts`：默认 true（子 Agent 不应弹确认框），除非 `shareAbortController`（说明是交互式子 Agent）

**Opt-in 共享**（给交互式子 Agent 用）：
- `shareSetAppState: true` → 用父的 setAppState
- `shareAbortController: true` → 共用同一个 abort controller（意味着可以弹 UI）
- `shareSetResponseLength: true` → metrics 归入父

### 为什么 readFileState 是 clone 而不是新建？

这是一个微妙的缓存一致性问题。父 Agent 的历史消息包含 `tool_use_id`，子 Agent 的 `forkContextMessages` 也包含这些同样的 id。`contentReplacementState` 根据这些 id 决定"这个工具结果要不要持久化到磁盘"。如果子 Agent 用全新的 state，它可能对同一个 id 做出**不同的决策**（因为它没见过之前的上下文） → API 请求中该 tool_result 的内容不同 → cache 前缀不一致 → cache miss。clone 保证做出**完全相同的决策**。

## 子 Agent 的分类：六种用途

### 1. Fork（隐式 fork）
- **触发**：模型调用 `AgentTool` 时不指定 `subagent_type`，且 `FORK_SUBAGENT` feature 开启
- **特点**：继承父的**完整上下文**（全部消息、全部工具 `tools: ['*']`），maxTurns=200
- **缓存技巧**：所有 fork 子 Agent 的历史消息前缀完全一致（用 `buildForkedMessages` 构建）：
  - 完整复制父的 assistant message（所有 tool_use block）
  - 为每个 tool_use 填充**完全相同的** placeholder result（`"Fork started — processing in background"`）
  - 只有最后一个 text block（directive）不同
  - 结果：99% 前缀命中 cache
- **权限**：`permissionMode: 'bubble'` — 权限请求冒泡到父终端
- **反递归保护**：`isInForkChild()` 检测消息中是否有 `<fork-boilerplate>` 标签，有则拒绝再 fork
- **与 coordinator 互斥**：`isCoordinatorMode()` 为 true 时 fork 被禁用

### 2. Explore Agent（只读代码搜索）
- **系统 prompt**：严格只读模式（READ-ONLY MODE），明确禁止一切写操作
- **禁用工具**：`AgentTool`（不能递归）、`FileEdit`、`FileWrite`、`NotebookEdit`、`ExitPlanMode`
- **模型**：内部用户 `inherit`（跟父一样），外部用户 `haiku`（追求速度）
- **无 maxTurns**：根据 thoroughness 自行决定搜多久
- **不加载 CLAUDE.md**：探索 Agent 不需要项目规则（主 Agent 有，会解读结果）

### 3. Plan Agent（规划专家）
- **系统 prompt**：软件架构师角色，只读，输出实现计划 + 关键文件列表
- **禁用工具**：同 Explore，外加 `AgentTool`
- **输出要求**：结尾必须列 3-5 个关键文件路径

### 4. Verification Agent（对抗性测试）
- **系统 prompt**：极长的对抗性验证指令——"你的工作不是确认它能用，是试图打破它"
- **设计哲学**：反向思维，检测实现者（另一个 LLM）可能遗漏的边界
- **工具**：可以用 Bash（包括写 /tmp 临时测试脚本），不能改项目文件
- **特殊能力**：如果有浏览器自动化 MCP（playwright/chrome），会主动使用

### 5. General Purpose Agent（通用子 Agent）
- **工具**：`tools: ['*']`（全部工具）
- **用途**：Skill/Command 的默认执行 Agent
- **模型**：未指定，走 `getDefaultSubagentModel()`

### 6. Session Memory Extraction Agent（后台记忆提取）
- **工具**：仅 `FileReadTool` + `FileEditTool`（限 session-memory 目录）
- **maxTurns**：5（两轮搞定：批量读 + 批量写）
- **缓存共享**：是（通过 `CacheSafeParams`）
- **skipTranscript**：true（临时性工作，不需要 session resume）
- **触发**：`postSamplingHook`，每轮 Agent turn 结束后

## AgentTool：模型怎么启动子 Agent

AgentTool 是一个普通的 `buildTool({...})` 工具，模型通过调用它来启动子 Agent。

**输入 schema**：
- `prompt`：给子 Agent 的任务描述
- `description`：3-5 词简短描述
- `subagent_type`（可选）：指定 Agent 类型（Explore/Plan/Verification/general-purpose）。不指定 + FORK_SUBAGENT 开启 → 走隐式 fork
- `model`（可选）：覆盖模型选择
- `run_in_background`（可选）：后台异步执行
- `mode`（可选）：权限模式
- `isolation`（可选）：`worktree`（git worktree 隔离）或 `remote`

**Agent 查找流程**：
1. 内置 Agent：Explore / Plan / Verification / general-purpose / fork
2. 用户自定义 Agent：从 `.claude/agents/` 目录加载的 Markdown 定义
3. Plugin Agent：通过插件系统加载的 Agent

## 子 Agent 的清理：不是结束了就完事

`runAgent.ts` 的 finally 块做了一系列清理：
- 断开 Agent 专属的 MCP 服务器连接
- 清除 frontmatter 注册的 session hooks
- 清除 prompt cache 追踪
- **释放 readFileState 缓存内存**（`.clear()`）
- 释放 fork 上下文消息数组（`.length = 0`）
- 取消 perfetto 追踪注册
- 清理 transcript 目录映射
- 清理 App State 里的 todos 条目
- **Kill 该 Agent 启动的所有后台 bash 任务**（`killShellTasksForAgent(agentId)`）

最后一点特别重要——如果子 Agent 启动了后台 shell 命令但自己退出了，这些命令必须被杀掉，否则变僵尸进程。

## Coordinator Mode vs Fork：两种并行策略

这两个**互斥**（`isCoordinatorMode() ? 禁用 fork : 正常`）。

| 维度 | Fork | Coordinator |
|---|---|---|
| **触发** | 模型省略 `subagent_type` | 环境变量 `CLAUDE_CODE_COORDINATOR_MODE` |
| **上下文** | 继承父的完整对话 + 工具 | 工人拿受限工具集 |
| **缓存** | 和父共享（字节级一致） | 不共享 |
| **权限** | bubble 到父终端 | 各工人独立 |
| **目标** | "帮我做这个子任务" | "把大任务分配给多个工人" |

Coordinator 的工人工具受限于 `ASYNC_AGENT_ALLOWED_TOOLS`（文件操作 + Bash + MCP，排除 AgentTool 内部工具），有 scratchpad 目录做跨工人的持久化知识共享。

## 设计启发

1. **缓存共享是架构决策的核心约束**：很多看起来奇怪的设计（为什么 clone 而不是新建 readFileState？为什么所有 fork 填相同的 placeholder？为什么不能随便设 maxOutputTokens？）都是为了保证 API 请求前缀字节一致从而命中 prompt cache。省下的钱是真金白银（90% token 走 cache_read 价格降 10 倍）
2. **状态隔离的粒度控制**：不是简单的"全隔离"或"全共享"，而是逐字段决定——tasks 注册穿透、denial 计数本地化、UI 回调 no-op、abort 单向传播。每个决策背后都有具体的 bug 故事
3. **sideQuery 和 runForkedAgent 的分界线**：需要工具？多轮？usage 追踪？→ runForkedAgent。只需要问一个问题拿答案？→ sideQuery。前者重（完整循环），后者轻（单次调用）
4. **子 Agent 的类型不是继承树**：六种子 Agent 不继承同一个基类，而是六个不同的配置对象传给同一个 `query()` 函数。和工具系统的 `buildTool()` 思路一致——接口统一，配置差异化
5. **清理要主动，不能靠 GC**：MCP 连接、bash 进程、transcript 映射、file state 缓存——每一个都需要显式清理。finally 块比 try 块更重要
