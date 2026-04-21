# Hermes Agent vs OpenClaw 对比分析

> 撰写日期：2026年4月21日

---

## 一、项目概览

### Hermes Agent

| 属性 | 详情 |
|------|------|
| **开发者** | [Nous Research](https://nousresearch.com/)（Hermes/Nomos/Psyche 模型背后的研究实验室） |
| **仓库** | [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent) |
| **Stars** | ~106k |
| **语言** | Python 87.6%, TypeScript 8.2% |
| **许可证** | MIT |
| **Slogan** | "The agent that grows with you" — 与你一起成长的 agent |
| **定位** | 自我改进的 AI Agent，由模型训练团队构建，内置学习闭环 |

### OpenClaw

| 属性 | 详情 |
|------|------|
| **创建者** | [Peter Steinberger](https://steipete.me/)（后加入 OpenAI）及社区 |
| **仓库** | [openclaw/openclaw](https://github.com/openclaw/openclaw) |
| **Stars** | ~361k |
| **语言** | TypeScript 89.9%, Swift 4.9% |
| **许可证** | MIT |
| **Slogan** | "THE AI THAT ACTUALLY DOES THINGS" — 真正干活的 AI |
| **定位** | 个人 AI 助手，运行在你自己的设备上，24/7 常驻 |

---

## 二、它们分别是什么样的 Agent

### OpenClaw — 个人 AI 助手 / "个人操作系统"

OpenClaw 的核心理念是做一个**本地优先、24/7 常驻、全平台接入**的个人 AI 助手。它不是编程工具，不是聊天机器人，而更像一个**数字员工/管家**。

关键特征：
- **Gateway 架构**：运行一个本地 Gateway 进程作为控制平面，管理会话、通道、工具和事件
- **全通道覆盖**：WhatsApp、Telegram、Discord、Slack、Signal、iMessage、微信、QQ、飞书、钉钉等 25+ 平台
- **本地运行，数据自控**：所有数据（记忆、技能、配置）都在用户自己的机器上，不依赖云端
- **可编程/自我编程**：agent 可以自己创建 skill（技能脚本），自己编写扩展
- **Companion Apps**：有 macOS 菜单栏应用、iOS/Android 节点应用
- **Live Canvas**：agent 可驱动的可视化工作区
- **Voice Wake**：支持唤醒词和连续语音

**典型使用场景**：
- 通过 Telegram 让 agent 帮你处理邮件、日历、航班
- 从手机上发消息让 agent 在你的 Mac 上跑代码、提 PR
- 设置 cron 定时任务做每日简报、监控数据
- 控制智能家居、WHOOP 健康追踪器
- 批量退订邮件、处理保险索赔

### Hermes Agent — 自我进化的 AI Agent

Hermes Agent 的核心理念是做一个**可学习、可自我改进**的 AI Agent。它由 AI 模型研究实验室 Nous Research 打造，强调的不只是"能用"，而是"越用越好"。

关键特征：
- **学习闭环（Closed Learning Loop）**：这是 Hermes 最核心的差异化特性
  - Agent 主动记忆（定期 nudge 自己保存知识）
  - 复杂任务后自主创建 skill
  - Skill 在使用中自我改进
  - FTS5 全文搜索 + LLM 总结实现跨会话召回
  - [Honcho](https://github.com/plastic-labs/honcho) 辩证式用户建模
- **多终端后端**：6 种终端后端 — local、Docker、SSH、Daytona、Singularity、Modal
- **Serverless 持久化**：Daytona/Modal 支持环境休眠，空闲时几乎零成本
- **消息平台覆盖**：Telegram、Discord、Slack、WhatsApp、Signal、Matrix、Mattermost、Email、SMS、钉钉、飞书、企业微信、Home Assistant 等 15+ 平台
- **研究就绪**：内置批量轨迹生成、Atropos RL 环境、轨迹压缩，可直接用于训练下一代工具调用模型
- **子 agent 并行**：可以 spawn 隔离的子 agent 做并行工作流

**典型使用场景**：
- 作为日常 AI 助手使用（与 OpenClaw 类似的个人助手场景）
- 开发、研究中作为自治 agent
- RL 训练数据生成和模型迭代
- 在低成本 VPS 或 GPU 集群上长期运行

---

## 三、功能对比表

| 功能维度 | OpenClaw | Hermes Agent |
|----------|----------|-------------|
| **定位** | 个人 AI 助手 / 数字员工 | 自我进化 AI Agent |
| **开发者** | 独立开发者 + 社区（Peter Steinberger） | AI 研究实验室（Nous Research） |
| **核心语言** | TypeScript | Python |
| **Stars** | ~361k | ~106k |
| **消息平台** | 25+ 平台（含微信、QQ、飞书） | 15+ 平台（含钉钉、飞书、企业微信） |
| **技能系统** | Skills + ClawHub 技能市场 | Skills + agentskills.io 开放标准 |
| **记忆系统** | 持久记忆，SOUL.md 人格文件，上下文持续 | 主动记忆 nudge + FTS5 搜索 + Honcho 用户建模 |
| **学习能力** | 可自创 skill、热加载 | **核心差异** — 闭环学习、skill 自我改进、定期知识持久化 |
| **定时任务** | Cron + Webhooks + Gmail Pub/Sub | 内置 cron，支持跨平台投递 |
| **沙箱/隔离** | Docker 沙箱、SSH、OpenShell | Docker、SSH、Daytona、Singularity、Modal |
| **Serverless** | 需自行部署 | Daytona/Modal serverless 持久化 |
| **语音** | Voice Wake、Talk Mode | 语音模式（CLI、Telegram、Discord VC） |
| **Companion Apps** | macOS App、iOS、Android | 纯 CLI + 消息平台（无原生 App） |
| **可视化** | Live Canvas (A2UI) | 无专属可视化 |
| **MCP 支持** | 是 | 是 |
| **多 Agent** | 多 agent 路由（隔离 workspace） | 子 agent 并行 |
| **RL/研究** | 无 | **核心差异** — Atropos RL、轨迹压缩、批量训练 |
| **模型支持** | Anthropic、OpenAI、本地模型 | Nous Portal、OpenRouter(200+)、OpenAI、NVIDIA NIM、小米MiMo、Kimi、MiniMax、HuggingFace |
| **互迁移** | — | `hermes claw migrate` 可从 OpenClaw 迁移 |

---

## 四、Hermes Agent 的技术新意

### 1. 闭环学习系统（Closed Learning Loop）

这是 Hermes Agent 最核心的创新。传统 AI agent（包括 OpenClaw）的"记忆"主要是被动的：用户告诉它记住某事，或系统自动保存对话摘要。而 Hermes 的记忆是**主动的、自我驱动的**：

- **Memory Nudge（记忆推动）**：agent 会定期自我提醒，将重要信息从短期上下文持久化到长期记忆中。这不需要用户手动触发。
- **Skill Auto-creation（技能自动创建）**：当 agent 完成一个复杂的多步骤任务后，它会自主分析这个过程并提炼成一个可复用的 skill。
- **Skill Self-improvement（技能自我改进）**：已创建的 skill 在后续使用中会被 agent 评估并优化。如果某个 skill 执行效果不佳，agent 会自动修订它。
- **Cross-session Recall（跨会话召回）**：FTS5 全文搜索引擎 + LLM 总结，让 agent 能从过去的所有对话中提取相关经验。

**与 OpenClaw 对比**：OpenClaw 也有持久记忆和 SOUL.md，也能自创 skill，但它的学习是被动的、一次性的——创建了就创建了，不会自我迭代优化。Hermes 实现了一个从"经验→记忆→技能→改进"的完整正反馈循环。

### 2. Honcho 辩证式用户建模

Hermes 集成了 [Honcho](https://github.com/plastic-labs/honcho) 做用户建模。不同于简单的"用户偏好记录"，Honcho 使用辩证法框架构建用户的深层理解模型——不只记录你说了什么，还推理你为什么这样说、你在什么情境下有什么偏好。随着交互增加，这个用户模型会越来越"理解你"。

### 3. 研究级 RL 训练管线

Hermes Agent 不仅是一个产品，还是一个**研究工具**：

- **Atropos RL 环境**：内置强化学习环境，可以将 agent 的使用轨迹作为训练数据
- **轨迹压缩（Trajectory Compression）**：将多步交互轨迹压缩为高效的训练样本
- **批量轨迹生成（Batch Trajectory Generation）**：可以大规模生成训练数据
- 这与 [OpenClaw-RL](https://github.com/Gen-Verse/OpenClaw-RL) 项目形成互补——OpenClaw-RL 是第三方为 OpenClaw 构建的 RL 训练框架，而 Hermes 是 **原生内置** RL 能力

**技术意义**：Nous Research 作为模型训练实验室，将 agent 的使用数据直接反馈到模型训练中，形成 `模型→agent→用户交互→训练数据→更好的模型` 的飞轮效应。这是一个独特的端到端闭环。

### 4. Programmatic Tool Calling（程序化工具调用）

Hermes 的 `execute_code` 工具允许 agent 编写 Python 脚本，通过 RPC 直接调用其他工具。这意味着 agent 可以将复杂的多步骤管线压缩成**单次推理调用**，大幅降低上下文成本和延迟。

### 5. 六种终端后端 + Serverless 持久化

- Local、Docker、SSH、Daytona、Singularity、Modal
- Daytona 和 Modal 支持 **serverless 持久化**：agent 的环境在空闲时休眠，有请求时自动唤醒
- 这使得 agent 可以在"几乎零成本"的状态下 24/7 待命

### 6. 自进化框架（hermes-agent-self-evolution）

Nous Research 还开源了 [hermes-agent-self-evolution](https://github.com/NousResearch/hermes-agent-self-evolution)，使用 DSPy + GEPA（遗传进化提示算法）来自动优化 Hermes 的技能、提示词和代码。这是 **agent 元优化** 的方向——不只是 agent 自己改进，而是有一个外层系统来进化 agent 本身。

---

## 五、架构差异总结

```
┌─────────────────────────────────┐    ┌─────────────────────────────────┐
│         OpenClaw                │    │        Hermes Agent             │
│                                 │    │                                 │
│  ┌──────────────┐               │    │  ┌──────────────┐               │
│  │   Gateway     │ ← 控制平面   │    │  │   Agent Loop  │ ← 推理循环   │
│  │  (TypeScript) │               │    │  │   (Python)    │               │
│  └──────┬───────┘               │    │  └──────┬───────┘               │
│         │                       │    │         │                       │
│  ┌──────┴───────┐               │    │  ┌──────┴───────┐               │
│  │   Channels    │ 25+ 平台    │    │  │ Messaging GW  │ 15+ 平台    │
│  └──────┬───────┘               │    │  └──────┬───────┘               │
│         │                       │    │         │                       │
│  ┌──────┴───────┐               │    │  ┌──────┴───────┐               │
│  │  Tools/Skills │               │    │  │ Tools/Skills  │               │
│  │  + ClawHub    │               │    │  │ + Skills Hub  │               │
│  └──────┬───────┘               │    │  └──────┬───────┘               │
│         │                       │    │         │                       │
│  ┌──────┴───────┐               │    │  ┌──────┴───────┐               │
│  │   Memory      │ 被动持久化   │    │  │   Memory      │ 主动学习     │
│  │  SOUL.md      │               │    │  │ + Nudge       │               │
│  │  MEMORY.md    │               │    │  │ + FTS5        │               │
│  └──────────────┘               │    │  │ + Honcho      │               │
│                                 │    │  └──────┬───────┘               │
│  ┌──────────────┐               │    │         │                       │
│  │ Companion Apps│ macOS/iOS/   │    │  ┌──────┴───────┐               │
│  │               │ Android      │    │  │   RL Pipeline │ ← 研究管线   │
│  └──────────────┘               │    │  │ Atropos/Batch │               │
│                                 │    │  └──────────────┘               │
└─────────────────────────────────┘    └─────────────────────────────────┘
```

---

## 六、结论

| 维度 | 选择 |
|------|------|
| **更成熟的个人助手体验** | OpenClaw — 更完整的 app 生态、更多通道、更大社区 |
| **更强的学习/自我改进能力** | Hermes Agent — 闭环学习是其核心卖点 |
| **AI 研究/模型训练** | Hermes Agent — 原生 RL 管线，模型训练团队背景 |
| **模型灵活性** | Hermes Agent — 200+ 模型通过 OpenRouter，含国产模型支持 |
| **原生 App 体验** | OpenClaw — macOS/iOS/Android companion apps |
| **从 OpenClaw 迁移** | Hermes 提供 `hermes claw migrate` 一键迁移 |

**总结**：OpenClaw 是目前最流行的开源个人 AI 助手（361k stars），以"24/7 常驻、全平台接入、本地自控"为核心卖点，偏向**产品/工程**方向。Hermes Agent 是 Nous Research 推出的"自我进化" agent（106k stars），在技术上的最大创新是**闭环学习系统**和**原生 RL 训练管线**，偏向**研究/自我改进**方向。两者都是开源、MIT 许可，且在快速融合——Hermes 已经提供了从 OpenClaw 的迁移工具，生态中的 skill 也在走向互通。
