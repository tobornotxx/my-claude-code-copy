# Claude Code 源码架构深度解析

> 本文档旨在帮助阅读者系统性地理解 Claude Code（一个基于终端的 AI 编程 Agent）的完整架构设计。

---

## 目录

1. [项目全景](#1-项目全景)
2. [启动流程（Boot Sequence）](#2-启动流程)
3. [Agent 核心循环架构](#3-agent-核心循环架构)
4. [上下文管理策略（Context Harness）](#4-上下文管理策略)
5. [工具系统（Tool System）](#5-工具系统)
6. [命令系统（Command System）](#6-命令系统)
7. [权限模型（Permission System）](#7-权限模型)
8. [状态管理（State Management）](#8-状态管理)
9. [Bridge / 远程执行架构](#9-bridge--远程执行架构)
10. [多 Agent 协调器（Coordinator Mode）](#10-多-agent-协调器)
11. [记忆系统（Memory / Memdir）](#11-记忆系统)
12. [服务层（Services）](#12-服务层)
13. [插件与技能扩展](#13-插件与技能扩展)
14. [UI 渲染层（Ink + React）](#14-ui-渲染层)
15. [成本追踪](#15-成本追踪)
16. [目录-功能映射速查表](#16-目录-功能映射速查表)

---

## 1. 项目全景

Claude Code 是一个**终端原生的 AI 编程 Agent 框架**，其设计核心可以概括为：

```
┌─────────────────────────────────────────────────────────┐
│                     用户终端 (TTY/PTY)                    │
│  ┌───────────────────────────────────────────────────┐  │
│  │              Ink (React for CLI) 渲染层             │  │
│  │    REPL Screen ← Components ← Hooks ← State      │  │
│  └───────────┬───────────────────────────┬───────────┘  │
│              │                           │              │
│  ┌───────────▼───────────┐  ┌───────────▼───────────┐  │
│  │     Command System    │  │      Query Engine      │  │
│  │  /commit /review ...  │  │  Agent Loop (Turn)     │  │
│  └───────────┬───────────┘  └───────────┬───────────┘  │
│              │                           │              │
│  ┌───────────▼───────────────────────────▼───────────┐  │
│  │              Tool Execution Layer                  │  │
│  │  FileRead│FileEdit│Bash│MCP│Agent│WebFetch│LSP... │  │
│  └───────────┬───────────────────────────┬───────────┘  │
│              │                           │              │
│  ┌───────────▼───────────┐  ┌───────────▼───────────┐  │
│  │   Permission Engine   │  │   Context Manager      │  │
│  │  规则·模式·审批·分类   │  │  记忆·Git·Compact·会话  │  │
│  └───────────────────────┘  └───────────────────────┘  │
│                                                         │
│  ┌─────────────────────────────────────────────────────┐│
│  │              Services Layer                         ││
│  │  API·Analytics·MCP·Compact·Session·Policy·Hooks    ││
│  └─────────────────────────────────────────────────────┘│
│                                                         │
│  ┌──────────────────────┐  ┌──────────────────────────┐ │
│  │    Bridge (Remote)   │  │   Coordinator (Multi-    │ │
│  │  HTTP 长轮询·CCR v2  │  │   Agent 多工作者并行)     │ │
│  └──────────────────────┘  └──────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

**技术栈要点：**
- **语言**: TypeScript (严格模式)
- **UI 框架**: Ink (React for terminal)，Yoga 布局引擎（src 内含纯 TS 移植版）
- **API**: Anthropic Claude API（流式调用）
- **进程模型**: 主线程 + Worker 子进程（Agent/Task）
- **扩展**: MCP 协议 + 插件系统 + 自定义技能

---

## 2. 启动流程

```
src/entrypoints/cli.tsx          ← 极速入口（1000ns 目标）
  │
  ├─ 快速路径: --version / --dump-system-prompt / --daemon-worker
  ├─ 并行预取: MDM 设置, Keychain 凭证
  │
  └─► src/main.tsx               ← 主初始化
        │
        ├─ OAuth / 策略 / Bootstrap 数据预取（并行）
        ├─ 信任对话（Trust Dialog）
        ├─ 模型选择 & 成本追踪初始化
        │
        └─► src/setup.ts          ← 运行时设置
              │
              ├─ Git 根目录检测
              ├─ Session ID 生成/恢复
              ├─ 迁移执行（src/migrations/）
              │
              └─► launchRepl()    ← 启动 REPL
                    │
                    └─► React App (Ink)
                          └─ REPL Screen + Message Loop
```

**关键文件：**

| 文件 | 职责 |
|------|------|
| [src/entrypoints/cli.tsx](src/entrypoints/cli.tsx) | CLI 入口，快速路径分发 |
| [src/main.tsx](src/main.tsx) | 主初始化逻辑，OAuth + 模型 + 会话 |
| [src/setup.ts](src/setup.ts) | Git 根检测、Session 初始化 |
| [src/replLauncher.tsx](src/replLauncher.tsx) | React/Ink REPL 启动器 |
| [src/entrypoints/init.ts](src/entrypoints/init.ts) | 项目初始化入口（`claude init`） |
| [src/entrypoints/mcp.ts](src/entrypoints/mcp.ts) | MCP Server 注册入口 |
| [src/migrations/](src/migrations/) | 启动时自动执行的幂等迁移 |

---

## 3. Agent 核心循环架构

这是整个框架中最关键的部分。Agent Loop 的核心实现分布在三个文件中：

### 3.1 QueryEngine — 主循环引擎

**文件**: [src/QueryEngine.ts](src/QueryEngine.ts)

QueryEngine 是整个 Agent 的"心脏"，实现了完整的 **Turn-based Agent Loop**：

```
用户输入
   │
   ▼
┌──────────────────────────────────────────┐
│ QueryEngine.executeTurn()                │
│                                          │
│  1. 消息规范化（去重、工具配对校验）        │
│  2. System Prompt 注入                    │
│     (用户上下文 + 记忆 + 附件)             │
│  3. ──► Claude API 流式调用 ◄──           │
│  4. 流式接收响应                          │
│     ├─ 纯文本 → 直接输出                  │
│     └─ tool_use → 拦截执行                │
│  5. 工具调用执行 + 结果注入                │
│  6. Post-sampling hooks                  │
│     (技能触发 / HFI 验证)                  │
│  7. 消息突变                              │
│     (记忆提取 / 内容替换)                  │
│  8. Token 预算检查 → 决定是否 Compact     │
│  9. 分析事件日志                          │
│                                          │
│  若有工具调用 → 回到步骤 3（下一轮）        │
│  若无工具调用 → 结束本次 Query             │
└──────────────────────────────────────────┘
```

**核心概念：**

- **Reactive Compact**: 实时监控上下文大小，自动判断何时需要压缩历史消息
- **Context Collapse**: 将旧消息批量合并为摘要
- **Microcompact**: 在每个 Turn 之间对工具结果做轻量级压缩
- **Tool Summary**: 将工具调用链合成为简短描述
- **Token Warning State**: 追踪缓存命中/创建令牌数和警告阈值
- **Fallback Trigger**: 捕获 API 令牌/速率限制错误并重试

### 3.2 Query — 单次查询编排

**文件**: [src/query.ts](src/query.ts)

```typescript
// 伪代码概括
async function executeQuery(userInput, context) {
  // 1. 构建 prompt chain
  const messages = buildPromptChain(systemPrompt, userContext, memory, attachments)
  
  // 2. 消息规范化
  normalizeMessages(messages) // 去重、工具配对校验
  
  // 3. 驱动单轮执行
  const result = await queryEngine.executeTurn(messages, { streaming: true })
  
  // 4. 收集指标
  accumulateUsage(result.tokens, result.cacheHits, result.duration)
  
  // 5. 处理拒绝/异常
  handlePermissionDenials(result)
  handleToolRejections(result)
}
```

### 3.3 Task — 任务抽象

**文件**: [src/Task.ts](src/Task.ts)

```typescript
type TaskType = 
  | 'local_bash'           // 本地 Bash 执行
  | 'local_agent'          // 本地子 Agent
  | 'remote_agent'         // 远程 Agent（bridge）
  | 'in_process_teammate'  // 进程内协作者
  | 'local_workflow'       // 本地工作流
  | 'monitor_mcp'          // MCP 监控任务
  | 'dream'                // 后台推演

type TaskStatus = 'pending' | 'running' | 'completed' | 'failed' | 'killed'
```

每个 Task 拥有独立的 `AbortController`、`AppState` 访问器和生命周期管理。

---

## 4. 上下文管理策略

上下文管理是 Agent 框架设计的核心难题。Claude Code 采用**多层上下文注入 + 自动压缩**的策略：

### 4.1 上下文组装

**文件**: [src/context.ts](src/context.ts)

```
System Prompt
  ├─ 基础指令（角色定义、安全规则）
  ├─ getSystemContext()
  │     ├─ Git 状态（branch, status, diffs）
  │     ├─ 最近提交
  │     └─ 用户名
  ├─ getUserContext()
  │     ├─ MEMORY.md（项目级记忆）
  │     ├─ Auto-memory（自动提取的记忆）
  │     ├─ 额外目录上下文
  │     └─ Team memory（团队共享记忆）
  ├─ 工具定义列表（动态过滤）
  └─ 会话附件
```

**缓存策略**: 上下文获取结果会被 memoize 缓存，只在 system prompt 注入时才刷新。

### 4.2 上下文压缩（Compact）

**文件**: [src/services/compact/](src/services/compact/)

这是 Claude Code 解决**长对话上下文溢出**的核心机制：

```
┌─────────────────────────────────────────┐
│            Compact 策略层级              │
│                                         │
│  Auto Compact (autoCompact.ts)          │
│    └─ 上下文接近 token 上限时自动触发     │
│                                         │
│  Reactive Compact (reactiveCompact.ts)  │
│    └─ 实时监控，动态决策（Feature-gated） │
│                                         │
│  Manual Compact (compact.ts)            │
│    └─ 用户通过 /compact 手动触发         │
│                                         │
│  Micro Compact (microCompact.ts)        │
│    └─ 每轮 Turn 间的轻量级工具结果压缩   │
│                                         │
│  Session Memory Compact                 │
│    (sessionMemoryCompact.ts)            │
│    └─ 会话记忆重建                       │
└─────────────────────────────────────────┘
```

**压缩流程**：
1. **分组** (`grouping.ts`)：将消息序列按逻辑分组
2. **摘要生成**：通过一个专用的 summarization agent 对旧消息生成摘要
3. **替换**：用摘要替换原始消息序列
4. **Post-compact hooks**：执行压缩后清理钩子
5. **警告管理** (`compactWarningHook.ts`)：追踪并展示压缩状态

### 4.3 会话历史

| 文件 | 场景 | 机制 |
|------|------|------|
| [src/history.ts](src/history.ts) | 本地粘贴历史 | 100 项队列，大块内容外部存储，通过引用 ID 检索 |
| [src/assistant/sessionHistory.ts](src/assistant/sessionHistory.ts) | 远程会话历史 | API 分页获取 `/sessions/{id}/events`，游标机制 |

### 4.4 Token 预算管理

QueryEngine 内建 token 预算监控：
- 每次 API 调用后统计输入/输出/缓存 token 
- 当总量接近模型上下文窗口时触发 Auto Compact
- 支持 "缓存友好" 的 prompt 排列（稳定前缀 → 动态后缀）

---

## 5. 工具系统

### 5.1 工具接口

**文件**: [src/Tool.ts](src/Tool.ts)

```typescript
interface Tool {
  name: string
  description: string
  inputSchema: JSONSchema       // 参数的 JSON Schema
  call(input, context): Promise<output>
  canUse?: CanUseToolFn         // 权限检查钩子
}
```

### 5.2 工具注册表

**文件**: [src/tools.ts](src/tools.ts) — 将所有工具汇总注册

### 5.3 工具分类

**文件**: [src/tools/](src/tools/) — 30+ 工具实现

| 类别 | 工具 | 说明 |
|------|------|------|
| **文件操作** | `FileReadTool`, `FileEditTool`, `FileWriteTool`, `GlobTool`, `NotebookEditTool` | 读/写/编辑文件，支持行范围、原子写入、格式化 |
| **代码执行** | `BashTool`, `REPLTool` | Shell 执行，stdin/stdout 捕获，超时控制 |
| **代码智能** | `LSPTool` | Go-to-definition, 查找引用, hover 信息 |
| **网络** | `WebFetchTool`, `WebSearchTool` | HTTP 请求、Tavily 搜索 |
| **子 Agent** | `AgentTool` | 生成子 Agent 执行隔离任务 |
| **MCP** | `MCPTool`, `ListMcpResourcesTool`, `ReadMcpResourceTool`, `McpAuthTool` | MCP 服务器发现/资源枚举/认证 |
| **用户交互** | `AskUserQuestionTool`, `SendMessageTool`, `TodoWriteTool` | 向用户提问、发消息、任务追踪 |
| **任务管理** | `TaskCreateTool`, `TaskUpdateTool`, `TaskListTool`, `TaskGetTool`, `TaskOutputTool` | 后台异步任务的创建/更新/查询/输出 |
| **规划模式** | `EnterPlanModeTool`, `ExitPlanModeV2Tool` | 切换 Plan / Execute 模式 |
| **工作树** | `EnterWorktreeTool`, `ExitWorktreeTool` | Git worktree 隔离环境 |
| **团队** | `TeamCreateTool`, `TeamDeleteTool` | 多 Agent 协作的团队管理 |
| **技能** | `SkillTool`, `BriefTool` | 调用用户定义技能、生成代码摘要 |
| **高级** | `MonitorTool`, `SleepTool`, `ScheduleCronTool`, `PushNotificationTool` | 进程监控/延时/定时触发/推送通知（Feature-gated） |

### 5.4 工具执行流程

```
Claude 输出 tool_use block
    │
    ▼
QueryEngine 拦截
    │
    ├─ toolMatchesName() 解析工具名
    ├─ canUse() 权限检查
    │    ├─ 允许 → 执行
    │    └─ 拒绝 → 返回权限拒绝消息
    ├─ tool.call(input, context) 执行
    ├─ 进度追踪（BashProgress, MCPProgress 等）
    ├─ 结果注入 tool_result 配对消息
    └─ 继续下一轮 Agent Loop
```

---

## 6. 命令系统

### 6.1 命令定义

**文件**: [src/commands.ts](src/commands.ts) — 命令注册表

```typescript
interface Command {
  type: 'prompt' | 'action' | 'inline'
  name: string
  description: string
  usage?: string
  getPromptForCommand(args, context): Promise<ContentBlock[]>
  allowedTools?: string[]    // 限制该命令可用的工具
  progressMessage?: string
}
```

**命令类型**：
- **prompt**: 生成 prompt 注入给 Claude 处理（如 `/commit`）
- **action**: 直接执行操作（如 `/clear`）
- **inline**: 内联执行（如 `/compact`）

### 6.2 命令目录

**文件**: [src/commands/](src/commands/) — 100+ 命令实现

| 分类 | 命令 | 说明 |
|------|------|------|
| **初始化** | `init`, `init-verifiers`, `install` | 项目设置、验证器安装 |
| **版本控制** | `commit`, `commit-push-pr`, `review`, `branch`, `diff` | Git 操作一站式 |
| **代码分析** | `advisor`, `insights`, `security-review`, `brief` | 代码审查与分析 |
| **配置** | `config`, `color`, `vim`, `voice`, `keybindings`, `output-style` | 个性化设置 |
| **记忆/历史** | `memory`, `export`, `backfill-sessions` | 知识管理 |
| **工作空间** | `session`, `tasks`, `plan`, `context`, `add-dir` | 会话与上下文管理 |
| **集成** | `mcp`, `skills`, `plugins`, `install-github-app`, `install-slack-app` | 外部扩展 |
| **调试** | `doctor`, `debug-tool-call`, `heapdump`, `ant-trace`, `env` | 诊断与调试 |
| **Bridge** | `bridge`, `bridge-kick` | 远程控制 |
| **Agent 高级** | `ultraplan`, `bughunter`, `good-claude` | 高级 Agent 工作流 |

### 6.3 命令执行流程

```
用户输入 "/commit fix typo"
    │
    ├─ 解析器匹配命令名/别名
    ├─ 命令 handler 构建 prompt 内容
    │    └─ 例如 commit.ts: 获取 git status/diff/recent commits
    ├─ allowedTools 限制可用工具集
    ├─ prompt 注入 Agent Loop
    └─ 结果流式返回
```

---

## 7. 权限模型

**文件**: [src/utils/permissions/](src/utils/permissions/) — 24 个文件

### 7.1 权限模式

```typescript
type PermissionMode = 'auto' | 'manual' | 'bypass'
```

- **manual** (默认): 每次敏感操作都需要用户确认
- **auto**: 匹配规则自动批准
- **bypass**: 跳过所有权限检查（危险模式）

### 7.2 多层权限检查

```
工具调用请求
    │
    ├─ 1. PermissionRule 规则匹配 (CLAUDE_PERMISSIONS.md 配置)
    │      └─ 路径模式、命令模式、正则匹配
    │
    ├─ 2. YOLO 分类器 (yoloClassifier.ts)
    │      └─ ML 分类器判断命令危险程度
    │
    ├─ 3. 危险模式检测 (shadowedRuleDetection.ts)
    │      └─ 检测规则冲突
    │
    ├─ 4. 拒绝追踪 (denialTracking.ts)
    │      └─ 记录被拒绝的操作
    │
    └─ 5. 用户审批工作流
           └─ 终端 UI 确认对话框
```

### 7.3 关键文件

| 文件 | 职责 |
|------|------|
| `PermissionRule.ts` | 规则定义与结构 |
| `PermissionMode.ts` | 模式管理 |
| `permissionRuleParser.ts` | 从配置文件解析规则 |
| `yoloClassifier.ts` | 自动模式下的安全分类器 |
| `denialTracking.ts` | 拒绝记录管理 |

---

## 8. 状态管理

### 8.1 AppState Store

**文件**: [src/state/](src/state/) — 自定义 Zustand 风格 Store

```typescript
interface AppState {
  messages: Message[]               // 当前对话消息
  tasks: Record<TaskId, TaskState>  // 活跃任务
  mainLoopModel: ModelSetting       // 当前使用的模型
  verbose: boolean                  // 详细输出模式
  effortLevel: 'fast' | 'thorough' // 努力级别
  fastModeEnabled: boolean          // 快速模式
  toolPermissionContext: PermissionContext // 权限上下文
  isAutoMode: boolean               // 自动模式
  speculations: SpeculationState    // 推测预取状态
  promptSuggestion?: { text, promptId }  // 提示建议
  // ...更多字段
}
```

**特性：**
- 基于订阅的更新：只在字段变化时重新渲染
- Selector hooks: `useAppState(s => s.messages)`
- 外部通知回调: `onChangeAppState`

### 8.2 Bootstrap State (全局单例)

**文件**: [src/bootstrap/state.ts](src/bootstrap/state.ts)

```typescript
// 一次性初始化的全局状态
{
  originalCwd: string          // 启动时的工作目录
  projectRoot: string          // Git root
  sessionId: SessionId         // 持久会话令牌
  isRemoteMode: boolean        // 是否 Bridge 模式
  isMainThread: boolean        // 主线程 vs Worker
  kairosActive: boolean        // 助手模式
  totalCostUSD: number         // 累计成本
  modelUsage: Record<model, Usage>  // 模型级令牌追踪
}
```

### 8.3 React Context 层

**文件**: [src/context/](src/context/) — 多个 React Context Provider

| Context | 职责 |
|---------|------|
| `stats.tsx` | 成本追踪存储 |
| `mailbox.tsx` | 组件间消息传递 |
| `notifications.tsx` | Toast 通知 |
| `overlayContext.tsx` | 覆盖层渲染（plan mode, dialogs） |
| `promptOverlayContext.tsx` | Prompt 预览覆盖层 |
| `fpsMetrics.tsx` | 性能监控 |
| `voice.tsx` | 语音模式上下文 |
| `QueuedMessageContext.tsx` | 后台消息队列 |

---

## 9. Bridge / 远程执行架构

Bridge 是 Claude Code 的**远程控制层**，实现了 Desktop ↔ Cloud 的双向会话管理。

### 9.1 架构图

```
┌──────────────┐         HTTP Long-Poll         ┌──────────────┐
│   Desktop    │ ◄══════════════════════════════► │  Cloud VM    │
│   (Client)   │     ingress: 消息轮询            │  (Session    │
│              │     egress:  结果流              │   Runner)    │
│  bridgeUI.ts │                                 │ bridgeMain.ts│
└──────────────┘                                 └──────┬───────┘
                                                        │
                                                 ┌──────▼───────┐
                                                 │  Bash 子进程   │
                                                 │ (隔离环境)     │
                                                 └──────────────┘
```

### 9.2 核心组件

| 文件 | 职责 |
|------|------|
| [src/bridge/types.ts](src/bridge/types.ts) | `WorkSecret` 类型定义：会话令牌、API URL、Git 信息、环境变量 |
| [src/bridge/bridgeApi.ts](src/bridge/bridgeApi.ts) | HTTP 客户端：OAuth 刷新、401 重试、可信设备令牌 |
| [src/bridge/bridgeMain.ts](src/bridge/bridgeMain.ts) | 长轮询 Worker 生命周期、会话生成、Worktree 管理 |
| [src/bridge/replBridge.ts](src/bridge/replBridge.ts) | 双向消息传输：ingress 轮询 + egress 结果流 |
| [src/bridge/replBridgeTransport.ts](src/bridge/replBridgeTransport.ts) | V1 (JSON) / V2 (二进制) 传输协议 |
| [src/bridge/sessionRunner.ts](src/bridge/sessionRunner.ts) | 子进程生成（隔离环境、超时管理） |
| [src/bridge/jwtUtils.ts](src/bridge/jwtUtils.ts) | JWT 令牌刷新调度 |
| [src/bridge/trustedDevice.ts](src/bridge/trustedDevice.ts) | 可信设备令牌管理 |
| [src/bridge/workSecret.ts](src/bridge/workSecret.ts) | CCR v2 SDK URL 构建 |
| [src/bridge/capacityWake.ts](src/bridge/capacityWake.ts) | 容量唤醒信号 |
| [src/bridge/pollConfig.ts](src/bridge/pollConfig.ts) | 自适应轮询间隔 |

### 9.3 WorkSecret 数据结构

```typescript
interface WorkSecret {
  version: number
  session_ingress_token: string
  api_base_url: string
  sources: Array<{ type: string, git_info?, ... }>
  auth: Array<{ type: string, token: string }>
  environment_variables?: Record<string, string>
  mcp_config?: unknown
  use_code_sessions?: boolean  // CCR v2 选择器
}
```

### 9.4 会话生命周期

```
WorkSecret 到达
  │
  ├─ 1. 解析凭证与源码信息
  ├─ 2. 创建隔离环境（可能使用 Git worktree）
  ├─ 3. 生成 Bash 子进程
  ├─ 4. 开始 ingress 轮询（接收指令）
  ├─ 5. 执行任务，流式回传 egress 结果
  ├─ 6. 监听控制请求（pause/resume/cancel）
  └─ 7. 完成/超时 → 优雅关闭（SIGTERM → SIGKILL）
```

---

## 10. 多 Agent 协调器

**文件**: [src/coordinator/](src/coordinator/)

### 10.1 协调器模式 (Feature-gated: `COORDINATOR_MODE`)

```
┌───────────────────┐
│   Coordinator     │
│   (主 Agent)      │
│                   │
│  ┌──────┐ ┌──────┐
│  │Worker│ │Worker│  ← 通过 AgentTool 生成
│  │  A   │ │  B   │
│  └──┬───┘ └──┬───┘
│     │        │
│     ▼        ▼
│  受限工具集:
│  - 文件工具（限定目录范围）
│  - Bash 执行
│  - MCP 资源
│  - 团队消息通道
└───────────────────┘
```

**特性：**
- Worker 在隔离的 Scratch Directory 中工作
- 工具集被限制，防止 Worker 越权操作
- 团队消息通道用于结果收集
- 会话模式追踪（恢复时保持 Coordinator/Normal 模式）

---

## 11. 记忆系统

**文件**: [src/memdir/](src/memdir/)

### 11.1 记忆类型与层级

```
项目根目录/
  └─ MEMORY.md              ← 项目级记忆（自动加载到上下文）
  └─ .claude/
       └─ memories/          ← 结构化记忆目录

~/.claude/
  └─ MEMORY.md              ← 用户全局记忆
```

### 11.2 核心模块

| 文件 | 职责 |
|------|------|
| `paths.ts` | 记忆文件路径解析 |
| `memoryTypes.ts` | 记忆类型定义（frontmatter 标签） |
| `memoryScan.ts` | 扫描项目层级中的记忆文件 |
| `findRelevantMemories.ts` | 语义匹配查找相关记忆 |
| `memoryAge.ts` | 记忆新鲜度分析 |
| `teamMemPaths.ts` | 团队记忆路径 |
| `teamMemPrompts.ts` | 团队记忆 prompt 模板 |

### 11.3 自动记忆提取

QueryEngine 的 Post-sampling hook 中包含**自动记忆提取**：
- 分叉一个子 Agent 分析对话
- 提取有价值的知识片段
- 持久化到 MEMORY.md

---

## 12. 服务层

**文件**: [src/services/](src/services/)

### 12.1 API 通信

| 文件 | 职责 |
|------|------|
| `api/claude.ts` | Anthropic SDK 调用/流式封装 |
| `api/withRetry.ts` | 指数退避自动重试 |
| `api/filesApi.ts` | 会话文件上传/下载 |
| `api/bootstrap.ts` | 会话初始化数据获取 |
| `api/grove.ts` | Grove API 高级搜索 |

### 12.2 分析与遥测

| 文件 | 职责 |
|------|------|
| `analytics/growthbook` | Feature Flag 管理 |
| `analytics/firstParty` | 事件上报 |
| `analytics/datadog` | APM 遥测 |

### 12.3 MCP 集成

| 文件 | 职责 |
|------|------|
| `mcp/client.ts` | MCP 服务器发现与工具注册 |
| `mcp/types.ts` | 协议定义 |
| `mcp/officialRegistry.ts` | 官方工具 URL 预取 |

### 12.4 策略与限制

| 文件 | 职责 |
|------|------|
| `policyLimits/` | API 速率限制、每轮工具调用限制、订阅级配额 |

---

## 13. 插件与技能扩展

### 13.1 插件系统

**文件**: [src/plugins/](src/plugins/)

```
.claude/plugins/
  └─ my-plugin/
       └─ index.ts    ← 编译后由 initBuiltinPlugins() 加载
```

**能力：** Hook 注册、工具定义、命令扩展、设置集成

### 13.2 技能系统

**文件**: [src/skills/](src/skills/)

```
~/.claude/skills/myskill.md
格式: frontmatter (name, description) + markdown 指令
调用: /myskill 或通过 SkillTool
```

- 内置技能库在 `src/skills/bundled/`
- MCP 技能构建器 (`mcpSkillBuilders.ts`) 可自动从 MCP 资源生成技能

---

## 14. UI 渲染层

### 14.1 技术栈

- **Ink**: React for Terminal（声明式终端 UI）
- **Yoga Layout**: Flexbox 布局引擎（`src/native-ts/yoga-layout/` 含纯 TS 移植版）

### 14.2 核心组件

**文件**: [src/components/](src/components/)、[src/screens/](src/screens/)

- `App.tsx` — React 根组件，全局 Context Provider
- `screens/REPL.tsx` — 主 REPL 界面（消息列表 + 输入区 + 状态栏）

### 14.3 Hooks 层

**文件**: [src/hooks/](src/hooks/) — 104 个 React hooks

覆盖：UI 状态、语音、导航、会话管理、快捷键、权限、遥测、工具集成。

关键 hooks:
- `useMainLoopModel` — 管理活跃 Claude 模型
- `useRemoteSession` — 远程执行上下文
- `useQueueProcessor` — 工具队列管理
- `useCancelRequest` — 查询取消集成
- `useCanUseTool` — 权限检查
- `useVoice` — 按住说话 STT

### 14.4 快捷键系统

**文件**: [src/keybindings/](src/keybindings/)

YAML/JSON 配置 → 解析 → 和弦支持 → 上下文感知动作分发

### 14.5 输出样式

**文件**: [src/outputStyles/](src/outputStyles/)

从项目目录加载自定义 CSS/ANSI 主题文件。

---

## 15. 成本追踪

**文件**: [src/cost-tracker.ts](src/cost-tracker.ts)、[src/costHook.ts](src/costHook.ts)

```typescript
// 追踪的指标
{
  totalCostUSD: number              // 会话累计花费
  totalAPIDuration: number          // API 调用耗时
  totalToolDuration: number         // 工具执行耗时
  totalLinesAdded: number           // 新增代码行数
  totalLinesRemoved: number         // 删除代码行数
  totalInputTokens: number          // 输入 token 数
  totalOutputTokens: number         // 输出 token 数
  totalCacheReadInputTokens: number // 缓存命中 token
  totalCacheCreationInputTokens: number // 缓存写入 token
  totalWebSearchRequests: number    // 网络搜索次数
}
```

**定价模型**: `src/utils/modelCost.ts` — 按模型分别计算输入/输出费率，含缓存折扣。

---

## 16. 目录-功能映射速查表

### 核心框架

| 路径 | 功能 | 重要度 |
|------|------|--------|
| `src/main.tsx` | 主入口，OAuth/模型/会话初始化 | ⭐⭐⭐ |
| `src/QueryEngine.ts` | Agent Loop 主循环引擎 | ⭐⭐⭐ |
| `src/query.ts` | 单次查询编排 | ⭐⭐⭐ |
| `src/Task.ts` | 任务类型/状态抽象 | ⭐⭐⭐ |
| `src/Tool.ts` | 工具接口定义 | ⭐⭐⭐ |
| `src/tools.ts` | 工具注册表 | ⭐⭐⭐ |
| `src/commands.ts` | 命令注册表 | ⭐⭐ |
| `src/context.ts` | 上下文组装（Git + 记忆） | ⭐⭐⭐ |
| `src/setup.ts` | 运行时初始化（Git 根、Session ID） | ⭐⭐ |

### 工具实现

| 路径 | 功能 |
|------|------|
| `src/tools/` | 30+ 工具的具体实现 |
| `src/tools/shared/` | 工具共享工具函数（权限、进度、错误格式化） |

### 命令实现

| 路径 | 功能 |
|------|------|
| `src/commands/` | 100+ 斜杠命令的实现 |
| `src/commands/commit.ts` | Git commit 工作流 |
| `src/commands/review.ts` | 代码审查 |
| `src/commands/init.ts` | 项目初始化 |
| `src/commands/ultraplan.tsx` | 高级规划 Agent |

### 状态管理

| 路径 | 功能 |
|------|------|
| `src/state/` | AppState Store（Zustand 风格） |
| `src/bootstrap/state.ts` | 全局单例状态（CWD, Session, 成本） |
| `src/context/` | React Context Providers（6+个） |

### 上下文与记忆

| 路径 | 功能 |
|------|------|
| `src/memdir/` | 记忆目录系统（扫描/匹配/持久化） |
| `src/services/compact/` | 上下文压缩策略（Auto/Reactive/Micro） |
| `src/history.ts` | 粘贴历史管理 |
| `src/assistant/sessionHistory.ts` | 远程会话历史 |

### 远程与协调

| 路径 | 功能 |
|------|------|
| `src/bridge/` | 远程控制层（长轮询·CCR v2·会话管理） |
| `src/coordinator/` | 多 Agent 协调器 |
| `src/server/` | DirectConnect WebSocket 会话 |
| `src/remote/` | 远程模式辅助 |

### 服务层

| 路径 | 功能 |
|------|------|
| `src/services/api/` | Anthropic API 调用·重试·文件传输 |
| `src/services/compact/` | 上下文压缩引擎 |
| `src/services/mcp/` | MCP 协议集成 |
| `src/services/analytics/` | 分析·Feature Flag·APM |
| `src/services/policyLimits/` | 速率限制·配额管理 |

### 权限系统

| 路径 | 功能 |
|------|------|
| `src/utils/permissions/` | 24 文件：规则·模式·分类器·拒绝追踪 |
| `src/types/permissions.ts` | 权限类型定义 |

### UI 层

| 路径 | 功能 |
|------|------|
| `src/components/` | Ink React 组件库 |
| `src/screens/` | 屏幕级组件（REPL 主界面） |
| `src/hooks/` | 104 个 React hooks |
| `src/ink/` | Ink 框架扩展 |
| `src/keybindings/` | 快捷键系统 |
| `src/outputStyles/` | 输出主题样式 |

### 扩展系统

| 路径 | 功能 |
|------|------|
| `src/plugins/` | 插件加载/注册 |
| `src/skills/` | 技能系统（内置 + MCP 自动生成） |

### 基础设施

| 路径 | 功能 |
|------|------|
| `src/entrypoints/` | CLI/MCP/Init 入口点 |
| `src/migrations/` | 配置迁移（模型升级、设置合并） |
| `src/utils/` | 564+ 工具函数（文件/Git/Shell/Token/格式化...） |
| `src/types/` | 类型定义（权限/插件/命令/ID/日志） |
| `src/native-ts/` | 纯 TS 移植（Yoga 布局引擎、文件索引、色差计算） |
| `src/upstreamproxy/` | CCR 容器侧 HTTP 代理 |
| `src/cost-tracker.ts` | 成本追踪器 |
| `src/costHook.ts` | 成本 React Hook |
| `src/voice/` | 语音模式启用标志 |
| `src/vim/` | Vim 模式支持 |
| `src/buddy/` | Companion 精灵动画 |
| `src/cli/` | CLI I/O 传输层（NDJSON、结构化输出、远程 IO） |

---

## 推荐阅读顺序

如果你打算深入阅读源码，建议按以下顺序：

1. **先理解数据流**: `src/main.tsx` → `src/setup.ts` → `src/replLauncher.tsx`
2. **核心 Agent Loop**: `src/QueryEngine.ts` → `src/query.ts` → `src/Task.ts`
3. **工具框架**: `src/Tool.ts` → `src/tools.ts` → 选读几个 `src/tools/` 下的具体实现
4. **上下文策略**: `src/context.ts` → `src/services/compact/` → `src/memdir/`
5. **权限模型**: `src/utils/permissions/PermissionRule.ts` → `PermissionMode.ts`
6. **状态管理**: `src/state/` → `src/bootstrap/state.ts`
7. **命令系统**: `src/commands.ts` → 选读几个命令
8. **远程架构**: `src/bridge/types.ts` → `bridgeMain.ts` → `replBridge.ts`
9. **UI 层**: `src/screens/` → `src/components/` → `src/hooks/`
10. **扩展系统**: `src/plugins/` → `src/skills/` → `src/services/mcp/`
