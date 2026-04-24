# WikiLLM 调研：LLM 驱动的知识库编译系统

> 调研日期：2026年4月24日
> 调研来源：Andrej Karpathy 推文、GitHub 三个 WikiLLM 实现、Claude Code 源码
> 调研核心问题：WikiLLM 的设计框架是怎样的？它和 Claude Code 记忆/Skill 系统的关系？

---

## 一、起源：Karpathy 的 WikiLLM 理念

2026年4月3日，Andrej Karpathy 发布了一条关于"用 LLM 构建个人知识库"的推文（2029万次浏览、5.7万收藏），描述了他近期大量 token 消耗不再是操纵代码，而是**操纵知识**的实践。

### 1.1 核心框架

Karpathy 提出了一个六阶段的知识编译流水线：

```
┌──────────────────────────────────────────────────────────────────┐
│                    WikiLLM 六阶段流水线                           │
│                                                                  │
│  1. Data Ingest    源文档（文章/论文/代码/数据集/图片）→ raw/      │
│        │                                                         │
│        ▼                                                         │
│  2. Wiki Compile   LLM 增量"编译" → 交叉链接的 .md wiki          │
│        │              • 摘要、反向链接                             │
│        ▼              • 概念分类、文章写作                         │
│  3. IDE (Obsidian)  查看 raw 数据、编译后 wiki、可视化             │
│        │                                                         │
│        ▼                                                         │
│  4. Q&A            对 wiki 做复杂问答（~100篇/~40万词时效果好）   │
│        │              • 不需要 RAG，LLM 自动维护索引               │
│        ▼                                                         │
│  5. Output         渲染为 markdown / Marp 幻灯片 / matplotlib图  │
│        │              • 输出反哺回 wiki（正反馈循环）              │
│        ▼                                                         │
│  6. Linting        LLM 健康检查 → 不一致/缺失/新候选              │
│                                                                  │
│  Extra: 朴素搜索引擎等辅助工具（CLI → LLM → 大型查询）            │
│  Future: 合成数据 + 微调，让 LLM "记住"数据在权重里               │
└──────────────────────────────────────────────────────────────────┘
```

### 1.2 设计哲学

| 原则 | 说明 |
|------|------|
| **LLM 写、人类不碰** | "you rarely ever write or edit the wiki manually, it's the domain of the LLM" |
| **编译而非检索** | 知识在编译阶段就合成好了，查询时直接读成品 wiki，不是每次从 raw 重新检索（**不是 RAG**） |
| **正反馈循环** | 用户的探索和问答结果被归档回 wiki，知识"越用越多" |
| **Obsidian 前端** | 用 Obsidian Web Clipper 收集素材，用 Obsidian 查看 wiki + 图谱 + 可视化 |
| **自动维护索引** | LLM 自动维护索引文件和文档摘要，小规模下无需 fancy RAG |

Karpathy 原话的结尾：*"I think there is room here for an incredible new product instead of a hacky collection of scripts."*

---

## 二、两个 WikiLLM 开源实现

Karpathy 推文后，社区快速出现了两个基于 Claude Code 的实现：

| 项目 | Stars | 语言 | 技术路线 | 状态 |
|------|-------|------|---------|------|
| **wang-junjian/wikillm** | 8 | TypeScript | Claude Code Skills + 中文 wiki | 2026年4月，活跃 |
| **Berkay2002/wikillm** | 0 | TypeScript | Claude Code Plugin（最工程化） | 2026年4月，v0.2.5 |

### 2.1 wang-junjian/wikillm — 忠实 Karpathy 理念的中文实现

最忠实于 Karpathy 原始理念的实现，直接使用 **Claude Code Skills** 驱动整个流水线。

**目录结构**：
```
wikillm/
├── CLAUDE.md                  ← 项目级 Claude 指令（定义目录结构、核心原则）
├── raw/                       ← 源文档（文章、论文、代码）
│   └── assets/                ← 图片等资源
├── wiki/                      ← LLM 编译的 markdown wiki
│   ├── concepts/              ← 核心概念文章
│   ├── practices/             ← 实践指南
│   ├── visual/                ← 可视化内容
│   ├── queries/               ← 查询存档（正反馈循环的体现）
│   ├── assets/                ← 图像资源
│   ├── INDEX.md               ← 内容目录
│   └── Glossary.md            ← 术语表
├── skills/wikillm/            ← Claude Code 技能
│   ├── SKILL.md               ← 技能入口（任务路由）
│   └── references/            ← 子文档
│       ├── workflows.md       ← 增量编译流程
│       ├── qa.md              ← Q&A 流程
│       ├── standards.md       ← 质量标准
│       └── errors.md          ← 常见错误
├── web/                       ← Next.js Web 前端（替代/补充 Obsidian）
└── README.md
```

**SKILL.md 的任务路由设计**：

技能入口采用"先分类、再执行"的路由模式：

```
用户发起请求
     │
     ├─ raw/ 有新文件需要编译 ──→ references/workflows.md（增量编译）
     ├─ 用户问了一个问题 ──────→ references/qa.md（Q&A流程）
     └─ 需要健康检查 ──────────→ references/errors.md（Linting）
```

**核心原则**：
1. 人工不直接编写 Wiki，仅负责投放素材和发起查询
2. LLM 负责理解、重写、链接与维护
3. 用户探索和查询被归档回 wiki
4. 图像下载到本地以便 LLM 引用

**额外亮点**：提供了 Next.js Web 应用，支持在浏览器中查看知识库（markdown 渲染、Obsidian 风格 wiki 链接解析、侧边栏导航）。

### 2.2 Berkay2002/wikillm — 最工程化的 Claude Code Plugin 实现

做成了**完整的 Claude Code 插件**，通过 `npx wikillm` 一键脚手架，提供 7 个 slash command skills + 1 个 subagent。这是目前**最完整的工程化实现**。

#### 2.2.1 安装与分发

通过 Claude Code Plugin Marketplace 分发：

```bash
# 在 Claude Code 中安装插件
/plugin marketplace add Berkay2002/wikillm
/plugin install wikillm@wikillm
/reload-plugins

# 在终端中 scaffold 一个 vault
npx wikillm          # 个人知识库
cd my-project && npx wikillm  # 项目知识库
```

初始化向导询问：Name → Kind（personal/project）→ Mode（solo/team）→ Features（slides/reports/visualizations/web-clipper）→ Domain → Automation

#### 2.2.2 Vault 目录结构

```
<vault>/
├── CLAUDE.md              ← 该 KB 的 schema（LLM 自动生成，按配置定制）
├── raw/                   ← 投放原始素材
│   └── assets/
├── wiki/                  ← 编译后的 wiki 文章
│   └── _index/
│       ├── INDEX.md       ← 内容目录（按分类索引）
│       ├── TAGS.md        ← 标签目录
│       ├── SOURCES.md     ← raw/ → wiki/ 溯源映射
│       ├── RECENT.md      ← 最近20条变更
│       └── LOG.md         ← 操作日志
├── outputs/               ← 生成的幻灯片/报告/可视化
└── .obsidian/             ← Obsidian 工作区配置
```

#### 2.2.3 七个 Skills 详解

| Skill | 功能 | 复杂度 |
|-------|------|--------|
| `/wikillm:using-wikillm` | 新手引导——如何使用插件、检测 .kb/ vault、集成项目 CLAUDE.md | 低 |
| `/wikillm:ingest` | 编译 raw/ 新文件 → wiki 文章，含并行 subagent 机制 | **高** |
| `/wikillm:query` | 在 wiki 上做问答，自动选输出格式，支持查询结果反哺 | 中 |
| `/wikillm:lint` | 断链/孤岛页/缺 frontmatter/矛盾检测/图连通性/重复检测 | **高** |
| `/wikillm:obsidian-cli` | 终端控制 Obsidian（搜索/读写/图查询） | 低 |
| `/wikillm:marp-cli` | 从 wiki 内容生成幻灯片（PDF/PPTX/HTML） | 低 |
| `/wikillm:generate-schema` | 重新生成 vault 的 CLAUDE.md | 低 |

#### 2.2.4 Ingest Skill 深度解析

Ingest 是整个系统最复杂的 skill，包含完整的编译流水线：

**检测阶段**：
1. 读 `wiki/_index/SOURCES.md` 获取已处理文件列表
2. 列出 `raw/` 中所有文件（排除 `raw/assets/`）
3. 不在 SOURCES.md 中的 = 新文件

**处理策略分两条路径**：

```
新文件数量
    │
    ├─ 1-2 个 → 顺序处理（orchestrator 直接做）
    │    1. 识别文件类型 + 读源文档
    │    2. 提取概念/实体/主题
    │    3. 检查 wiki 中是否已有覆盖
    │    4. 创建新文章 / 更新已有文章 / 跳过
    │    5. 交叉链接 [[wikilinks]]
    │    6. 更新四维索引（INDEX/TAGS/SOURCES/RECENT）
    │    7. git commit（每个源文件一次）
    │
    └─ 3+ 个 → 并行处理（三阶段 orchestrator-worker 模式）
```

**并行处理的三阶段**：

```
Phase 1: Pre-scan（orchestrator 独占）
  1a. 清点每个源文件：文件类型、大小/复杂度、主题范围、关键概念
  1b. 读取当前 wiki 状态（SOURCES.md + INDEX.md）
  1c. 构建概念分配表（Concept Assignment Table）★核心
       • 列出所有源文件中出现的概念
       • 如果一个概念出现在多个源文件中 → 指定 ONE 个源文件为 owner
       • 其他源文件该概念标记为 link-only（只引用不创建）
  1d. 决定是否真的需要并行
       • 轻量任务 → orchestrator 自己做（省去 subagent 开销）
       • 3+ 实质性源文件 → 并行 dispatch

Phase 2: Dispatch Workers（并行）
  - 每个源文件一个 ingest-worker subagent
  - 全部并行启动
  - 每个 worker 收到：源文件路径 + 当前索引 + 概念分配表

Phase 3: Reconcile（orchestrator 独占）
  1. 处理失败的 worker
  2. 去重检查（文件名相同 or 标题近似的文章）
  3. 交叉链接审计（worker 之间新文章的互链）
  4. 聚合更新四维索引
  5. git commit + push
```

**概念分配表是避免重复的核心机制**——确保同一个概念只有一个 worker 负责创建/更新，其他 worker 只引用。

#### 2.2.5 ingest-worker SubAgent 详解

`agents/ingest-worker.md` 定义了并行编译的工作单元：

```yaml
name: ingest-worker
tools: Read, Write, Edit, Glob, Grep, Bash
model: sonnet                      # 用更便宜的模型
permissionMode: bypassPermissions  # 免授权
skills: ingest                     # 继承 ingest skill 的写作规范
memory: project                    # 读取项目级记忆
color: green                       # UI 区分标记
```

**Worker 的职责边界**：
- ✅ 读取源文件、识别概念、写/更新自己 OWN 的 wiki 文章、交叉链接
- ❌ 不创建/更新 LINK-ONLY 概念的文章
- ❌ 不更新 INDEX/TAGS/SOURCES/RECENT/LOG
- ❌ 不执行 git 操作

**返回格式**（供 orchestrator 聚合）：
```
SOURCE: raw/filename.ext
NEW_ARTICLES: [[article-1]], [[article-2]]
UPDATED_ARTICLES: [[existing-article]]
TAGS_USED: tag1, tag2, tag3
SUMMARY_PAGE: [[source-summary-page]]
```

#### 2.2.6 Lint Skill 深度解析

Lint 是知识库健康检查的核心，包含 **11 类检查**，按优先级排序执行：

| 检查 | 描述 | 自动修复? |
|------|------|----------|
| 2.0 Scoping | 定义过滤规则：排除 raw/ 噪音 + _index/ 自引用 | N/A |
| 2.1 断链 `[[wikilinks]]` | 检测 wiki/ 中指向不存在页面的链接 | ✅ 修复或降级为纯文本 |
| 2.2 缺失 Frontmatter | 检查 created/updated/tags/sources 四个必需字段 | ✅ 自动补全 |
| 2.3 孤岛/近孤岛页 | 0 或 1 个真实反向链接的文章 | ✅ 添加链接 |
| 2.4 缺失概念页 | 被多次引用但不存在的概念 → 创建 stub | ✅ 创建 stub |
| 2.5 缺失交叉引用 | 同 tag 的文章之间缺少互链 | ✅ 在 Related 节添加 |
| 2.6 矛盾 | 多篇文章对同一主题有冲突说法 | ❌ 仅记录，等人类审核 |
| 2.7 过期/未验证声明 | 带 `[unverified]` 标记的内容 | ✅ 联网验证 |
| 2.8 索引一致性 | wiki 文章 ↔ INDEX/SOURCES/TAGS 双向核对 | ✅ 补全缺失条目 |
| 2.9 Hub 检测 | 识别高反向链接的"枢纽"文章，提升索引排序 | ✅ 调整 INDEX 排序 |
| 2.10 图连通性 | 检测断开的连通分量（孤岛集群） | ✅ 添加桥接链接 |
| 2.11 重复/近重复审计 | H1 相似 / tag + source 重叠 / 出链 Jaccard ≥70% | ❌ 仅标记，等人类决策 |

**关键设计决策**：
- **Scoping 过滤**是最重要的 meta-规则——不做 scoping 的 lint 会产生 70+ 假阳性
- **矛盾和重复不自动修复**——记录到 LOG.md 并标记 `[contradiction]` / `[duplicate-candidate]`，等待人类决策
- **每次最多 50 个修复**——防止失控修改
- **修复后必须重新验证**——LOG 是操作历史的 source of truth，必须准确
- **Git 工作流**：单次 commit，`git add` 指定文件而非 `git add .`，绝不 stage `raw/` 文件

#### 2.2.7 Query Skill 深度解析

Query 的核心理念是**先图后搜、只读 wiki、选择性反哺**：

```
用户问题
    │
    ▼
1. Find（INDEX → 搜索 → 标签）
    │   • INDEX.md 可直接定位 → 直接 Read，不搜索
    │   • 模糊/跨领域问题 → obsidian search path="wiki"
    │   • 主题性问题 → 读 TAGS.md 按 tag 找
    │
    ▼
2. Traverse（反向链接遍历）★关键差异点
    │   • obsidian backlinks file="<primary>"
    │   • Hub 文章 = 包含跨领域 gotchas/对比表的核心页
    │   • 这一步是 Obsidian 优于纯 grep 的核心原因
    │
    ▼
3. Synthesize（综合回答）
    │   • 只读 wiki/，不读 raw/
    │   • 跟踪 [[wikilinks]] 获取上下文
    │   • 标注信息来源和查找路径
    │
    ▼
4. Output Format（选择输出格式）
    │   ├─ 简单回答 → 直接在对话中回复
    │   ├─ 结构化分析 → outputs/reports/
    │   ├─ 演示 → outputs/slides/（调用 marp-cli）
    │   └─ 数据可视化 → outputs/visualizations/（Python + matplotlib）
    │
    ▼
5. File Back?（选择性反哺）
    │   • 纯查找 → 不反哺（避免更新噪音）
    │   • 暴露矛盾/发现新联系/产出对比决策 → 反哺
    │
    ▼
6. Note Gaps（记录知识缺口）
        • 标记 [gap] 到 LOG.md
        • 建议用户往 raw/ 投放素材
```

#### 2.2.8 三种运行模式

| 模式 | Vault 路径 | 自动化 | Git 策略 |
|------|-----------|--------|---------|
| **Personal** | `~/.wikillm/<name>/` | 每日 ingest + 每周 lint（定时） | 私有 vault，独立 commit |
| **Project Solo** | `<repo>/.kb/` | 每日 ingest + 每周 lint（定时） | raw/ 提交，.obsidian/ gitignore |
| **Project Team** | `<repo>/.kb/` | 仅手动——ingest 前先 pull | raw/ + wiki/ 都提交，协调 push |

定时自动化需要 Claude Desktop 在计划触发时运行。

#### 2.2.9 文章写作规范

Ingest skill 定义了严格的文章规范：

```markdown
---
created: YYYY-MM-DD
updated: YYYY-MM-DD
tags: [tag1, tag2]
sources: [raw/filename.ext]
---

# Article Title

Brief summary (2-3 sentences).

## Content

Main content organized with headers.

## Related

- [[Related Article 1]]
- [[Related Article 2]]

## Sources

- `raw/source-file.md` — what was drawn from this source
```

**写作原则**：
- **百科式但可读**——为"忘了细节的未来的自己"写
- **一篇一概念**——如果一节可以独立为有用参考，就拆成独立文章
- **Warning Callout 是最高 ROI 产出**——把隐式陷阱变成显式警告

**Warning Callout 示例**（来自实际 wiki）：
```markdown
> ## Response Shape
>
> **IMPORTANT — `results`, not `urls`:** The response uses `results` for the
> URL array, not `urls`. This has caused bugs in multiple implementations.
```

---

## 三、WikiLLM 与 Claude Code 记忆/Skill 系统的关系

### 3.1 层级关系

两个 WikiLLM 实现**直接建立在 Claude Code 的 Skill 和记忆机制之上**。它们不是竞争关系，而是：

```
┌─────────────────────────────────────────────────────────────────┐
│                    Claude Code 底层平台                          │
│                                                                 │
│  ┌──────────────────┐    ┌───────────────────────┐              │
│  │  Memory System   │    │   Skill System        │              │
│  │  ─────────────   │    │   ────────────        │              │
│  │  MEMORY.md 索引  │    │   SKILL.md 入口       │              │
│  │  .md 记忆文件    │    │   references/ 子文档  │              │
│  │  user/feedback/  │    │   渐进式披露加载      │              │
│  │  project/ref 类型│    │   name+desc 匹配激活  │              │
│  └──────────────────┘    └───────────────────────┘              │
│           ↑                        ↑                            │
│           │                        │                            │
│  ┌────────┴────────────────────────┴────────────────┐           │
│  │              WikiLLM 应用层                       │           │
│  │                                                   │           │
│  │  CLAUDE.md  ← 项目指令（定义目录/规范/模式）      │           │
│  │  skills/    ← 7 个技能（ingest/query/lint/...）   │           │
│  │  agents/    ← subagent（ingest-worker）           │           │
│  │  hooks/     ← 事件钩子                            │           │
│  │                                                   │           │
│  │  raw/ ──→ LLM编译 ──→ wiki/ ──→ Q&A/可视化      │           │
│  │       Obsidian / Web 作为前端                      │           │
│  └───────────────────────────────────────────────────┘           │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 具体对应关系

#### (A) CLAUDE.md ↔ 项目级指令

| Claude Code 原生 | WikiLLM 用法 |
|------------------|-------------|
| `CLAUDE.md` 是项目级指令文件，每次 session 自动加载 | WikiLLM 生成定制化的 `CLAUDE.md` 到 vault 根目录，定义 KB 的目录结构、编译规范、运行模式 |
| 项目团队手写维护 | `npx wikillm` 根据用户选择自动生成，`/wikillm:generate-schema` 可重新生成 |

#### (B) Skills ↔ WikiLLM 的 7 个技能

WikiLLM 完全遵循 Claude Code 的 Skill 标准格式：

```yaml
# skills/ingest/SKILL.md
---
name: ingest
description: Use this skill when ingesting raw source files...
---
```

但 WikiLLM 的 skill 比典型的编码辅助 skill **复杂得多**：
- ingest skill 有 207 行（9.54KB），含并行调度策略、三阶段编译流水线
- lint skill 有 226 行（14.7KB），含 11 类检查规则和验证流程
- 使用 `references/` 子目录存放辅助文档（file-types.md 等）

#### (C) Agents ↔ ingest-worker subagent

WikiLLM 用 Claude Code 的 Agent 机制定义了 `ingest-worker`：

| 属性 | 值 | 说明 |
|------|-----|------|
| `model` | sonnet | 用更便宜的模型做 worker |
| `permissionMode` | bypassPermissions | 免授权 |
| `skills` | ingest | 继承写作规范 |
| `memory` | project | 读项目记忆 |
| `tools` | Read, Write, Edit, Glob, Grep, Bash | 受限工具集 |

这是 Claude Code Agent 机制的典型应用——orchestrator 用主模型做决策和协调，worker 用 sonnet 做批量执行。

#### (D) Memory ↔ WikiLLM 的知识存储

这是最本质的区别：

| 维度 | Claude Code Memory | WikiLLM Wiki |
|------|-------------------|-------------|
| **解决的问题** | "怎么做"——编程偏好、项目约定、反馈记录 | "知道什么"——领域知识的结构化存储和查询 |
| **知识来源** | 用户交互中的隐式反馈 | 用户主动投放的原始文档 |
| **存储粒度** | 单条记忆（一个 .md 文件，几行） | 整篇文章（交叉链接的 wiki 页面，可达数千字） |
| **索引方式** | `MEMORY.md` 单行指针，上限 200 行 / 25KB | 四维索引：INDEX + TAGS + SOURCES + RECENT |
| **加载策略** | 每次对话自动全量加载索引到 context | 查询时 LLM 按需读取相关 wiki 页面 |
| **冲突处理** | Last-write-wins | Lint 的矛盾检测（§2.6）→ 标记后等人类审核 |
| **知识演化** | 用户纠正 → 覆盖旧记忆 | 增量编译 + 定期 lint + 查询结果反哺 |
| **总量级** | 数十条记忆，KB 级 | 数百篇文章，数十万词，MB 级 |

**核心洞察**：
- Claude Code Memory = **工作记忆**（Working Memory）：session-to-session 的偏好持久化
- WikiLLM Wiki = **知识记忆**（Semantic Memory）：大规模领域知识的编译和查询
- 两者**互补而非竞争**

### 3.3 技术实现上的复用关系

WikiLLM（Berkay2002 版）使用了 Claude Code 的以下机制：

| Claude Code 机制 | WikiLLM 如何使用 |
|-----------------|-----------------|
| **Plugin System** | `.claude-plugin/plugin.json` + `marketplace.json` 实现分发和安装 |
| **Skill System** | 7 个 `skills/*/SKILL.md` 定义所有工作流 |
| **Agent System** | `agents/ingest-worker.md` 定义并行编译的 subagent |
| **Hooks** | `hooks/hooks.json` 定义事件钩子 |
| **CLAUDE.md** | vault 根目录的 `CLAUDE.md` 定义 KB schema |
| **Slash Commands** | 每个 skill 暴露为 `/wikillm:*` 命令 |
| **Subagent Dispatch** | ingest orchestrator 并行派发 ingest-worker subagent |
| **Git Integration** | ingest 和 lint 都有明确的 git commit 规范 |

---

## 四、设计启示与分析

### 4.1 "编译"而非"检索"的知识管理范式

WikiLLM 最大的设计创新是把知识管理类比为**编译器**：

```
传统 RAG:   raw docs → 向量化 → 每次查询时检索 → LLM 生成
WikiLLM:    raw docs → LLM 编译 → 成品 wiki → 直接读取回答
```

**编译的优势**：
- 知识合成（综合多个来源）在编译时完成，不是每次查询重做
- 交叉引用已经建好，矛盾已经标记
- 查询速度快——读成品 wiki，不需要向量检索
- 知识质量可以通过 lint 持续提升

**编译的代价**：
- 编译本身消耗大量 token
- 更新需要增量编译机制
- 编译质量取决于 LLM 的理解能力

### 4.2 Obsidian 是什么，以及它为什么重要

#### Obsidian 基础

Obsidian 是一个**本地 markdown 笔记软件**。它不联网、不用数据库，就是读你磁盘上某个文件夹（称为 vault）里的 `.md` 文件。它的核心卖点是把文件之间的引用关系构建成一张**知识图谱**，并提供图形化的 Graph View 来可视化文件间的链接关系。

#### Wikilink：纯文本的链接机制

WikiLLM 中文档之间的链接使用 Obsidian 扩展的 markdown 语法——**wikilink**，写法就是 `[[文件名]]`：

```markdown
# Attention Mechanism

Attention 是 [[Transformer]] 的核心组件，最早在 [[Seq2Seq]] 中被提出。

## Related
- [[Multi-Head-Attention]]
- [[Scaled-Dot-Product]]
```

这里的 `[[Transformer]]`、`[[Seq2Seq]]` 等就是链接。**它们是纯文本，不是代码，不是特殊格式**——就是用双方括号包一个文件名，对应 vault 下的 `Transformer.md`、`Seq2Seq.md`。

#### 正向链接 vs 反向链接

这是理解 WikiLLM 检索机制的关键概念：

| 类型 | 含义 | 存在位置 |
|------|------|---------|
| **正向链接** | 本篇 md 里写了 `[[X]]` → 本篇引用了 X.md | **写在文件内容里**，打开文件直接能看到 |
| **反向链接** | 别的 md 里写了 `[[本篇]]` → 谁引用了本篇 | **不写在任何文件里**，是 Obsidian 扫描全部文件后实时计算出来的 |

举个具体例子：

```
# attention-mechanism.md 的内容里写了：
...是 [[Transformer]] 的核心...

# 此时：
- attention-mechanism.md 的正向链接包含：Transformer.md（文件里写了 [[Transformer]]）
- Transformer.md 的反向链接包含：attention-mechanism.md（有人引用了我）

# 但你打开 Transformer.md，文件内容里不会出现 "attention-mechanism" 这几个字
# 反向链接只在 Obsidian 的侧边栏或 CLI 命令里才能查到
```

**简单说：正向链接是编译时 LLM 写在文件里的，反向链接是 Obsidian 运行时全局扫描算出来的。**

#### Obsidian 的图查询能力

Obsidian 启动时会扫描 vault 下所有 `.md` 文件，解析其中所有 `[[wikilink]]`，在内存中构建一张有向图（节点 = 每个 .md 文件，边 = A 文件里出现了 `[[B]]`）。基于这张图，Obsidian 提供了一组命令行 API（需开启 CLI 插件），WikiLLM 用这些命令让 agent 查图：

```bash
# 查反向链接：谁引用了这个文件
obsidian backlinks file="attention-mechanism"
# → transformer-architecture.md, bert-overview.md, vision-transformer.md

# 查正向链接：这个文件引用了谁
obsidian links file="attention-mechanism"
# → transformer.md, seq2seq.md, multi-head-attention.md

# 查孤岛页：没有任何反向链接的文件
obsidian orphans

# 查断链：引用了不存在的文件名
obsidian unresolved
# → [[some-concept]] 被引用了 3 次但没有对应的 .md 文件

# 全文搜索（可限定目录）
obsidian search query="attention" path="wiki"

# 按标签查文件
obsidian tags format=json
```

**如果没有 Obsidian**，这些能力就需要自己写脚本实现——比如反向链接，你得遍历所有 `.md` 文件，用正则 `\[\[target\]\]` 搜索。Obsidian 帮你维护了这张图，并提供了查询接口。

#### 反向链接对 Agent 的核心价值

正向链接是编译时 LLM 写死的 2-5 个"我觉得相关的"，反向链接是运行时全局扫描出的"所有实际引用了我的"。编译时 LLM 的判断不一定完整——它可能漏掉了某个后来新增的文章。但反向链接是实时算的，不会漏。

Agent 查 backlinks 本质上就是在问："**整个知识库里，还有谁在谈论这个概念？**"——这个问题靠读单个文件是答不了的。

反向链接最大的价值是**发现 hub 文章**（被很多文章引用的枢纽页面）。这些 hub 文章往往包含跨领域的对比表、gotchas、权威框架——是让回答"完整"的关键。但你从 INDEX 的平铺列表里看不出哪些是 hub，只有图遍历能发现。

Berkay2002 版的 query skill 原文：

> "Graph traversal is the single biggest advantage of querying Obsidian over flat-markdown grep. If you skip this step, you're leaving information on the table."

### 4.3 WikiLLM 的检索召回机制详解

#### Karpathy 本人说了多少？

关于检索，Karpathy 原推只有一段模糊描述：

> "I thought I had to reach for fancy RAG, but the LLM has been pretty good about auto-maintaining index files and brief summaries of all the documents and it reads all the important related data fairly easily at this ~small scale."

他说了"index files"（复数）和"summaries"，但**从未具体定义过 INDEX.md / TAGS.md / SOURCES.md / RECENT.md 这套四维索引**。四维索引结构是 Berkay2002 的实现设计。wang-junjian 版用的是 INDEX.md + Glossary.md，结构也不同。

**概念是 Karpathy 提出的（LLM 自动维护索引），具体结构是各实现者自己设计的。**

#### 检索本质：就是 Agent 读文件

WikiLLM 的检索**没有任何黑科技**——不用向量数据库，不用 embedding，不用 similarity search。本质上就是 agent 调用 `read_file` / `grep` 等工具，一篇一篇读 markdown 文件。**LLM 自己就是检索引擎**——读索引、做判断、读文件、跟链接。

这和 Claude Code 的记忆系统是同构的：都是"索引 + 文件内容"的两层结构，agent 通过读文件工具访问。

#### 分层检索路由

虽然底层都是 `read_file`，但检索有一套分层路由策略（来自 Berkay2002 版 query skill）：

```
用户问题
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│  Layer 1: INDEX.md 直查                                  │
│  读 INDEX.md（按分类的文章目录 + 一行摘要）              │
│  如果 ≤2 篇文章明显匹配 → 直接 Read 文件，跳过搜索     │
│  INDEX 本质上是人工策划的查找表（curated lookup table）  │
└────────────────┬────────────────────────────────────────┘
                 │ INDEX 不够定位
                 ▼
┌─────────────────────────────────────────────────────────┐
│  Layer 2: 全文搜索 / 标签搜索                            │
│  • obsidian search query="<term>" path="wiki"           │
│  • 或 grep（Obsidian 没运行时的 fallback）               │
│  • 主题性问题 → 读 TAGS.md 按 tag 找                    │
└────────────────┬────────────────────────────────────────┘
                 │ 找到主要目标文章
                 ▼
┌─────────────────────────────────────────────────────────┐
│  Layer 3: 图遍历（反向链接）                             │
│  obsidian backlinks file="<primary-article>"            │
│  发现"谁引用了这篇文章"→ 找到 hub 文章                  │
│  这一步是 Obsidian 优于纯 grep 的核心原因               │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│  Layer 4: 读文件 + 跟链接（多跳阅读）                    │
│  读 wiki 文章正文 → 看到 [[wikilinks]] → 读关联文章     │
│  每篇文章末尾的 ## Related 节提供 2-5 个正向链接         │
└─────────────────────────────────────────────────────────┘
```

每一步读完文章后，agent 从文件内容中的 `[[wikilink]]` 和 `## Related` 节看到相关文件名，然后决定是否继续读。反向链接则从 Obsidian CLI 获取，补充正向链接覆盖不到的全局关联。

**一句话总结**：WikiLLM = **有向图结构的 markdown 文件系统** + **LLM 维护的索引文件** + **agent 的 file read 工具做图遍历**。创新不在检索技术，而在"编译"这个动作——把检索的难度从"在杂乱原始文档里找信息"降低到"在整理好的百科里查目录、跟链接"。

### 4.4 规模天花板与已知局限

#### INDEX 全量加载的问题

WikiLLM 当前的检索方案有一个明确的**规模天花板**。每次查询时 agent 需要先读 INDEX.md 来定位起始文章。粗略估算：

| wiki 规模 | INDEX.md 大小 | 检索效果 |
|-----------|-------------|---------|
| ~100 篇 / ~400K 词 | ~10-15KB | ✅ Karpathy 说"效果不错" |
| ~500 篇 / ~2M 词 | ~50-75KB | ⚠️ 需要 TAGS 辅助 + 多轮搜索 |
| ~5000+ 篇 | ~500KB+ | ❌ INDEX 占大量 context，纯文件读取扛不住 |

Karpathy 自己也说了 "at this ~small scale"——他默认这只适用于小规模。当 INDEX.md 本身太长，agent 读完索引后留给实际工作的 context 就不够了。

#### 四维索引是否是原始设计？

需要明确：**四维索引（INDEX/TAGS/SOURCES/RECENT）是开源实现者的设计，不是 Karpathy 本人定义的**。Karpathy 只提到了"index files"和"summaries"，没有指定具体结构。两个实现者各自设计了不同的索引方案。

#### 目前社区没有标准解法

截至 2026 年 4 月，两个 WikiLLM 实现都是**纯文件读取**，没有引入向量检索。可能的原因：
1. 实际使用者的 wiki 规模都还在"小规模"范围
2. 引入向量检索会打破"纯 markdown + Obsidian"的简洁性
3. 200K context window 的模型已经能装下不少索引

但可以预见，当 wiki 规模增长后，大概率会出现 **INDEX.md 定位 → 向量检索缩小范围 → 精读文章** 的混合路径。

### 4.5 并行编译中的概念所有权

ingest 的并行处理引入了一个精巧的协调机制——**概念分配表**（Concept Assignment Table）：

```
概念 "Attention Mechanism"
  ├─ 出现在 raw/transformer-paper.md（深度讨论）→ OWNER
  ├─ 出现在 raw/bert-overview.md（简要提及）→ LINK-ONLY
  └─ 出现在 raw/nlp-survey.md（简要提及）→ LINK-ONLY
```

这解决了经典的**分布式写入冲突**问题——多个 worker 同时想创建同一篇文章。解决方式是在 orchestrator 阶段就做好**静态分配**，让每个 worker 只写自己 OWN 的概念，对其他概念只加 `[[wikilinks]]` 引用。

这种设计也呼应了记忆冲突调研中的思路：**冲突在写入时预防，而非读取时解决**。

---

## 五、总结

### WikiLLM 的本质

WikiLLM 是一个**知识编译系统**：把原始素材通过 LLM 编译成结构化、交叉链接的 wiki，以 Obsidian 为前端，以 Claude Code Skills 为自动化引擎。它不是 RAG 的替代品，而是一种新范式——**编译时合成，查询时直读**。

### 与 Claude Code 的关系

```
Claude Code Memory  = 工作记忆（偏好/约定，KB 级，自动加载）
WikiLLM Wiki        = 知识记忆（领域知识，MB 级，按需读取）
Claude Code Skills  = WikiLLM 的执行引擎（ingest/query/lint 流水线）
Claude Code Agents  = WikiLLM 的并行编译基础设施（ingest-worker）
Claude Code Plugins = WikiLLM 的分发渠道（marketplace）
```

两者互补：Memory 记住"怎么做"，WikiLLM 存储"知道什么"。WikiLLM 是 Claude Code 生态中第一个把 Skill/Agent/Plugin 三大机制综合运用来构建**非编程领域应用**的项目。
