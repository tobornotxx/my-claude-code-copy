# Claude Code Harness Engineering & Context Engineering 面试回答稿

> 适用问题："Claude Code 的 Harness Engineering 有哪些技巧？"、"在 Context Engineering 和 Harness Engineering 上有哪些设计？"、"讲一下你对 Agent 工程化的理解"。

---

## 开场总述（30 秒）

Claude Code 是 Anthropic 开源的终端原生 AI 编程 Agent，它的核心工程挑战不在于"怎么让模型更聪明"，而在于**如何用工程手段约束、引导和保护一个 Agent 的运行时行为**——这就是所谓的 Harness Engineering。我对它的源码做了比较深入的研读，主要有以下几个层面的设计让我印象深刻。

---

## 一、Agent Loop 的状态机设计

Claude Code 的 Agent 主循环是一个 `while(true)` 的 AsyncGenerator，大概 1750 行代码。它有两个核心设计：

**第一是可变状态与不可变参数的严格分离。** 所有运行时状态集中在一个 `State` 对象里，包括消息历史、工具上下文、轮次计数等等，每次 continue 时整体替换而不是逐字段赋值，降低状态不一致的风险。而不可变的配置参数——比如 system prompt、canUseTool 函数、fallback model——在循环外一次性解构，循环内绝不修改。

**第二是用 transition 标记实现显式状态机。** `state.transition` 记录了"上一次为什么做了 continue"，而不只是"要不要 continue"。它定义了 7 种 continue 原因，比如 `next_turn` 是正常工具调用完成，`collapse_drain_retry` 是 413 后提交压缩重试，`max_output_tokens_escalate` 是升级输出 token 限制。这种设计的好处是可以精确防止恢复死循环——比如 collapse drain 后重试仍然 413，检查 transition reason 就知道不能再走 drain 路径了，得降级到 reactive compact。每条恢复路径都有明确的入口和退出条件，可审计、可测试。

用 AsyncGenerator 作为流式控制原语也很精妙——循环通过 yield 逐条发射消息，消费方控制消费速度，天然实现背压控制和懒求值，而且生产和渲染完全解耦。

---

## 二、上下文预算管理——五层渐进式压缩流水线

这是整个系统最精密的 Harness 设计。Claude Code 的上下文管理不是一刀切压缩，而是**五层机制按代价递增顺序依次执行**，前面能搞定就不触发后面。

**第一层是 Tool Result Budget，零成本裁切。** 每个工具都有一个 `maxResultSizeChars` 属性，单个结果超过这个阈值时，持久化到磁盘文件，上下文中只保留路径引用。不调 API、不改结构，是最廉价的防线。但要注意，这里说的是"Tool Result Budget 这一层是否把结果存到磁盘"的阈值，和"读文件工具本身是否截断输出"是两回事。FileRead 工具自身是有截断的——默认单次读取上限 25000 tokens（约 256KB 文件大小），超过会报错要求用 offset/limit 分段读。但它的 `maxResultSizeChars` 设为 Infinity，意味着读取结果不管多大都留在上下文里、不走磁盘持久化。这是因为如果把读取内容持久化为磁盘文件再插个路径引用，模型看到路径后会再次调 FileRead 去读——触发"Read → 看到路径 → 再 Read"的死循环。

**第二层是 History Snip，精确删除噪声。** 模型可以通过 SnipTool 标记某些消息为噪声，然后从发给 API 的消息列表中**成对删除** assistant 的 tool_use 消息和 user 的 tool_result 消息（Claude API 协议要求 tool_use 和 tool_result 必须配对，工具结果按协议放在 user 消息槽位中），不做任何摘要。比如一个搜索工具返回 500 行结果，模型只用了其中 3 行，剩下 497 行就是纯噪声，直接从请求中去掉最划算。这种"直接不发"的方式会改变消息序列结构，导致 prompt cache 失效，但 Snip 触发频率很低（需要模型主动调用，一个会话可能只触发 0-2 次），所以偶尔一次缓存失效的成本可以接受。系统还会在 token 增长一定量但模型没主动清理时，注入提醒消息让模型行使这个权利。

**第三层是 Microcompact，缓存感知的定向删除。** 它的删除目标是特定类型工具的**旧**结果——具体来说是 FileRead、Shell（Bash）、Grep、Glob、WebSearch、WebFetch、FileEdit、FileWrite 这 8 类工具的 tool_result（删最旧的，保留最近 N 个）。这些工具的共同特点是结果体积大、时效性强（旧的搜索结果和旧的文件内容在后续轮次中价值急剧下降）。Microcompact 每轮迭代都自动执行，和 Snip 的偶发触发形成互补。而这一层最体现工程深度的地方在于：它根据 prompt cache 是否存活选择不同的实现路径来删这些旧结果。热缓存时用 API 的 `cache_edits` 能力——发给 API 的消息文本**一个字都不改**，tool_result 里的完整内容照样在请求体里，但请求中同时附带 cache_edits 指令告诉 API"这些 tool_result 对应的 KV cache 你别用了"。API 服务端收到后，推理时跳过这些 token 的 KV cache，模型生成下一个 token 时 attention 不会看到它们——效果等价于"发了但让模型假装没看到"。因为消息文本本身没动，cache key 不变，其余部分的 KV cache 全部命中。冷缓存时（距上次 assistant 消息超过 60 分钟）KV cache 已失效，那就直接把旧的 tool_result 内容物理清空为 `[Old tool result content cleared]` 占位符，只保留最近 5 个可压缩的工具结果。Snip 是"直接不发"，Microcompact 热路径是"发了但让 API 忽略"——前者更彻底（连带宽都省了，且可以删 tool_use+tool_result 整对），后者保护缓存（绝大多数轮次都走这条路径）。

**第四层是 Context Collapse，结构保留的归档。** 把旧的对话轮次归档为摘要，但保留轮次结构，类似 git log，每一轮独立总结。和全量压缩相比，它保留了更多结构化信息。

**第五层是 Autocompact，全量摘要兜底。** 这层又分两条路径。首先尝试 SM-Compact，它直接读取后台 Session Memory 子 Agent 已经维护好的笔记文件作为摘要，压缩时刻不调 API，毫秒级完成；如果 SM-Compact 不可用，才走 Full Compact，fork 子 Agent 调 API 做全量摘要，耗时 5-10 秒。SM-Compact 的本质是**成本前移**——把生成摘要的 API 成本从压缩时刻分散到日常后台维护中。

还有一个被动防线叫 **Reactive Compact**——不主动压缩，等 API 真返回 413 再处理。通过"暂扣错误消息"实现对用户透明的恢复：流式循环中检测到 413 但不立即 yield 给 UI，先尝试恢复；恢复失败才暴露错误。

整个流水线的核心原则是：**代价递增、信息损失递增、触发频率递减。能用便宜的解决就不用贵的。**

---

## 三、工具系统的约束与编排

Claude Code 有 40 多个工具，全部用纯函数式的 `buildTool()` 工厂函数构建，零继承。工具系统的 Harness 设计有几个亮点：

**Fail-Closed 默认值。** 所有工具的默认配置选择最保守的选项：并发安全默认 false、只读默认 false、破坏性默认 false。新工具如果忘了设置，不会被意外并发执行或跳过权限检查。必须主动声明才能放宽——安全约束的默认方向是"收紧"。

**流式工具并行执行。** 模型还在流式输出时就开始执行已收到的工具调用，不等说完。通过 StreamingToolExecutor 实现：只读工具之间可以并行，遇到写操作严格串行。结果按接收顺序缓冲输出，保证不会乱序。这就像 CPU 的指令流水线——不等一条指令完全完成就开始下一条，最小化端到端延迟。

**错误级联的精确隔离。** Bash 命令出错时取消所有并行兄弟工具（因为 bash 命令间有隐式依赖），但不取消主循环——用独立的 siblingAbortController 精确控制影响范围。其他工具出错不级联。

**Deferred Loading——按需加载工具 Schema。** 40 多个工具的 schema 如果全部发给 API，会占用大量 prompt token。解决方案是核心工具始终加载，低频工具只发名字列表，模型需要时通过 ToolSearch 拿到完整 schema 再调用。这和数据库索引的思路一样——轻量索引始终可见，重量级内容按需加载。

**工具池排序保证 cache 稳定性。** 所有工具合并后按名称排序再发给 API，保证 prompt cache key 每次一致。如果工具顺序不确定，API 会认为是不同的 prompt，缓存全废。

---

## 四、权限与安全 Harness

一个工具从模型输出到实际执行要过 **六道关**：工具查找（支持别名回退）→ Zod schema 类型校验 → 自定义业务校验 → Pre-Hook（外部钩子可拦截/修改输入）→ 权限决策 → 实际执行（带 OpenTelemetry tracing）。这是典型的深度防御。

每次权限决策都记录来源——hook 批准、会话级临时规则、用户设置规则、交互确认——永远可回溯。

BashTool 作为最危险的工具，施加了最严格的多维约束：命令解析加安全分类、macOS 用 sandbox-exec / Linux 用 seccomp 沙箱隔离、读写分类决定并发安全性、sed -i 特殊处理为编辑预览模式。

FileEditTool 实现了 **TOCTOU 防护**——validate 阶段检查文件新鲜度，call 阶段执行前再检查一次，因为两个阶段之间存在时间窗口。Windows 上还有时间戳精度问题，所以时间戳对比失败后做内容对比兜底，是双重验证。

子 Agent 遵循**最小权限原则**：记忆提取 Agent 只有只读工具加 memory 目录写权限；Compact Agent 只有 FileRead 和 ToolSearch；Coordinator Worker 的文件工具限定目录范围、无 MCP。

---

## 五、记忆系统——写入-索引-召回闭环

Claude Code 的记忆系统是一个完整的"写入 → 索引 → 召回 → 注入 → 新鲜度管理"的闭环，不只是把对话存下来。

**索引-内容分离。** MEMORY.md 是索引文件，每行是一个指向具体记忆文件的指针加一句话摘要，始终加载到 user context（上限 200 行 / 25KB）。具体记忆文件的内容按需通过语义召回路径加载。这和数据库索引思路完全一致——不需要把全部数据塞进上下文，只需要一个足够小的索引来决定"需要加载哪些"。

**三条召回路径分工明确：** Path A 是静态注入——系统说明加 MEMORY.md 索引，粗粒度、零成本、始终可用；Path B 是 CLAUDE.md 延迟加载——只有模型读到某个目录下的文件时才加载该目录的指令文件；Path C 是语义召回——异步预取，用 Sonnet 从 frontmatter 清单中选出最多 5 个最相关的记忆文件注入。越精细的路径成本越高但精度也越高。

**异步预取利用自然等待时间。** 用户输入后立刻异步启动 Path C 预取，模型开始输出和工具执行期间（通常 5-30 秒）预取并行完成，工具完成后直接消费结果，用户感知延迟为零。

**写入端的互斥与合并。** 主 Agent 和后台提取 Agent 可能同时写记忆，用互斥检测加合并缓冲实现轻量级协调——不加锁（太重），而是如果主 Agent 这轮已经写了记忆，后台提取就跳过；提取进行中又来新请求就暂存上下文，当前完成后尾随运行。

**新鲜度管理。** 超过 1 天的记忆会附带验证提醒："This memory is 47 days old. Verify against current code before asserting as fact." 承认记忆会过时，不阻止使用但提醒验证。还有会话级节流——累计召回不超过 60KB，compact 后自然重置。

---

## 六、错误恢复与韧性设计

Claude Code 的错误恢复不是简单的 try-catch，而是精心设计的多级渐进恢复。

**暂扣-恢复模式。** 流式接收 API 响应时检测到可恢复错误（413、max_output_tokens）不立即 yield 给 UI，而是"暂扣"。恢复成功则用户完全无感知，恢复失败才把错误暴露。API 层面的错误不一定是终端用户层面的错误。

**Max Output Tokens 渐进升级。** 输出命中 token 限制时，先从默认 8K 升级到 64K；还不够就注入"Resume directly — no recap"消息让模型续写，最多 3 次；3 次后硬停。不是断崖式失败，而是多级恢复的渐进过程。

**413 的三级恢复链。** 先 Context Collapse drain（提交暂存摘要）→ 再 Reactive Compact（触发压缩）→ 都失败才报错。用 transition 标记防止在恢复链中循环。

**Autocompact 熔断器。** 连续失败 3 次后停止尝试。经典的熔断器模式——持续失败就停下来，给系统恢复时间，避免资源浪费。

---

## 七、性能 Harness——缓存与并行

**Prompt Cache 稳定性工程。** 多处设计都是为了保护 prompt cache 命中率：工具按名称排序保证 cache key 一致、热缓存时用 cache_edits 不改消息内容、system prompt 设置 cache_control 确保稳定前缀。cache 命中率直接影响 API 调用成本和延迟。

**Forked Agent 的缓存共享。** 子 Agent（记忆提取、compact 摘要）通过 runForkedAgent 运行，共享主对话的 system prompt 和消息前缀，API 端可以复用主对话的 KV cache。子 Agent 不是"额外的成本"，通过缓存共享大幅降低增量开销。

**多路异步预取。** 循环入口处并行启动记忆预取和技能发现预取，利用工具执行的自然等待时间并行完成。不是"减少等待"，而是"在等待中做有用的事"。

---

## 八、生命周期钩子与 Prompt Engineering

**Stop Hooks——模型说完不等于真的完了。** 模型输出完成且没有工具调用时触发 stop hook 链：模板分类、prompt 建议、记忆提取、auto-dream 等。如果 stop hook 返回阻塞错误（比如验证器发现代码错误），Agent Loop 会注入错误消息让模型修复。在"模型认为已完成"和"系统确认已完成"之间插入了验证层。

**Token Budget 续写。** 当模型在 budget 未用完时就停下了，注入 meta 消息促使继续输出；检测到边际递减时提前停止。既控制不要生成太少，也不要生成无意义内容。

**附件系统的精确注入时机。** 不同上下文信息在不同时机注入——system prompt 在循环开始、压缩在 API 调用前、记忆和技能在工具执行后。信息注入不是"全部塞进去"，而是在最合适的时机注入最相关的信息。

---

## 总结：十大设计原则（收尾用）

如果要总结的话，Claude Code 的 Harness Engineering 体现了十大设计原则：

1. **渐进式降级**——先用最轻量的手段，不行再升级
2. **Fail-Closed 默认值**——安全约束默认收紧，显式 opt-in 放宽
3. **显式状态追踪**——关键决策点都有可审计的追踪
4. **成本前移**——贵的操作尽可能提前做或并行做
5. **索引-内容分离**——轻量索引始终可见，重量级内容按需加载
6. **精确隔离**——错误和权限的影响范围被精确限定
7. **利用自然等待时间**——在等待中做有用的事
8. **双重验证**——关键路径上有冗余安全措施
9. **模型自主性加系统兜底**——给模型决策权但不完全信任
10. **缓存感知设计**——所有修改消息内容的操作都考虑对 prompt cache 的影响

这些原则不只适用于 Claude Code，对任何工程化的 Agent 系统都有借鉴意义。谢谢。
