# AI Agent Skills 全面调研

> 撰写日期：2026年4月21日

---

## 一、Skills 是什么

### 概念定义

**Skills（技能）** 是 AI Agent 生态中的一种**轻量级扩展机制**。每个 skill 本质上是一个包含 `SKILL.md` 文件的文件夹，通过 YAML frontmatter 定义元数据，通过 Markdown 正文定义指令，可选地包含脚本、模板和参考文档。

Agent Skills 已经成为一个**开放标准**，由 Anthropic 最初开发并发布，目前被 20+ 主流 AI agent/IDE 产品采纳。

```
my-skill/
├── SKILL.md          # 必需：元数据 + 指令
├── scripts/          # 可选：可执行代码
├── references/       # 可选：参考文档
└── assets/           # 可选：模板、资源
```

### 核心格式

`SKILL.md` 最小格式：

```yaml
---
name: pdf-processing
description: Extract PDF text, fill forms, merge files. Use when handling PDFs.
---

# PDF Processing

## When to use this skill
Use this skill when the user needs to work with PDF files...

## How to extract text
1. Use pdfplumber for text extraction...
```

**必需字段**：
- `name`：1-64 字符，小写字母+数字+连字符
- `description`：1-1024 字符，描述功能和使用时机

**可选字段**：
- `license`：许可证
- `compatibility`：环境要求
- `metadata`：扩展元数据（如 OpenClaw 特有的 gating 配置）
- `allowed-tools`：预授权工具列表（实验性）

---

## 二、Skills 的工作原理

Skills 采用**渐进式披露（Progressive Disclosure）** 机制来高效管理上下文：

```
┌─────────────────────────────────────────────────────────┐
│  1. 发现（Discovery）                                    │
│     Agent 启动时只加载每个 skill 的 name + description   │
│     (~100 tokens/skill)                                  │
│                                                          │
│  2. 激活（Activation）                                    │
│     当任务匹配某 skill 的 description 时，                │
│     Agent 读取完整 SKILL.md 到上下文                     │
│     (建议 < 5000 tokens)                                 │
│                                                          │
│  3. 执行（Execution）                                    │
│     Agent 按指令执行，按需加载 scripts/ references/      │
│     assets/ 中的文件                                     │
└─────────────────────────────────────────────────────────┘
```

这种设计让 agent 在保持快速的同时，按需获取更多上下文。

### 为什么用纯文本而不是代码？

Skills 的关键设计哲学是**指令优先**（instruction-first）：

- **自文档化**：人类和 AI 都能直接阅读理解
- **可审计**：安全审查只需读 Markdown
- **可移植**：纯文件，易于编辑、版本控制、分享
- **跨平台**：同一个 skill 可以在 Claude Code、OpenClaw、Hermes、Copilot 等不同 agent 上运行

---

## 三、AgentSkills 开放标准

### 标准概况

- **发起者**：Anthropic
- **官网**：[agentskills.io](https://agentskills.io/)
- **GitHub**：[agentskills/agentskills](https://github.com/agentskills/agentskills)
- **状态**：开放标准，接受社区贡献

### 采纳产品（20+）

| 产品 | 类型 |
|------|------|
| **Claude Code** | Anthropic 的 agentic 编程工具 |
| **OpenAI Codex** | OpenAI 的编程 agent |
| **GitHub Copilot** | GitHub 的 AI 编程助手 |
| **Gemini CLI** | Google 的命令行 AI 工具 |
| **OpenClaw** | 开源个人 AI 助手 |
| **OpenHands** | 开源 AI 软件工程平台 |
| **Amp** | Sourcegraph 的 AI 编程工具 |
| **Roo Code** | AI 编程助手 |
| **Kiro** | AWS 的 AI 编程工具 |
| **Junie** | JetBrains 的 AI 编程 agent |
| **TRAE** | 字节跳动的 AI IDE |
| **OpenCode** | 开源编程 agent |
| **Firebender** | AI 编程工具 |
| **Piebald / Emdash / Agentman / ...** | 其他兼容产品 |

### Skills 能做什么

- **领域专业知识**：法律审查流程、数据分析管线等专业知识打包成可复用指令
- **新能力扩展**：创建演示文稿、构建 MCP 服务器、分析数据集
- **可重复工作流**：将多步骤任务变成一致、可审计的工作流
- **跨产品互通**：同一个 skill 在不同 agent 产品中复用

---

## 四、Skills 生态平台

### ClawHub — OpenClaw 的 Skill 市场

- **网址**：[clawhub.ai](https://clawhub.ai/)
- **规模**：57,300+ skills，180,000+ 用户，12,000,000+ 下载
- **安装方式**：`openclaw skills install <skill-slug>` 或 `clawhub install <slug>`
- **安全扫描**：VirusTotal + OpenClaw 内置扫描，安装前自动检查
- **评分**：平均 4.8 分

### Skills Hub (agentskills.io) — Hermes 的 Skill 市场

- **网址**：[agentskills.io](https://agentskills.io/)
- **与 Hermes Agent 深度集成**，也兼容其他 AgentSkills 标准产品
- **Hermes 特色**：skill 可以在使用中自我改进

### 其他 Skill 来源

- **GitHub 仓库**：许多 skill 作者直接在 GitHub 发布
- **项目内嵌**：放在 `<workspace>/skills/` 或 `<workspace>/.agents/skills/` 中
- **全局共享**：放在 `~/.agents/skills/` 或 `~/.openclaw/skills/` 中

---

## 五、最热门 Skills 深度分析

以下数据基于 ClawHub 2026年4月21日的下载排行。

### 1. self-improving-agent（403k 下载，3.3k stars）

| 属性 | 详情 |
|------|------|
| **作者** | @pskoett |
| **功能** | 捕获学习、错误和纠正，实现持续改进 |
| **许可** | MIT-0 |

**工作原理**：

这是目前最火的 skill。它在项目中创建一个 `.learnings/` 目录，包含三个核心文件：
- `LEARNINGS.md`：纠正、洞察、知识盲区、最佳实践
- `ERRORS.md`：命令失败和集成错误
- `FEATURE_REQUESTS.md`：用户请求的功能

**核心机制**：
1. **自动检测触发**：当用户纠正 agent（"不对，应该是..."）、命令失败、知识过时等情况自动记录
2. **结构化日志**：每条记录有 ID（`LRN-YYYYMMDD-XXX`）、优先级、状态、区域标签
3. **递归模式检测**：自动搜索已有条目，关联相似问题（`See Also`），复发时提高优先级
4. **晋升机制**：当学习具有广泛适用性时，从 `.learnings/` 晋升到项目级记忆（`CLAUDE.md`、`AGENTS.md`等）
5. **Hook 集成**：通过 `UserPromptSubmit` 和 `PostToolUse` hook 自动注入提醒
6. **自动 Skill 提取**：当某个学习反复出现且已验证，可自动提取为新的独立 skill

**创新点**：实现了 "学习→记录→检测模式→晋升→提取新 skill" 的完整正反馈循环。

---

### 2. Self-Improving + Proactive Agent（169k 下载，981 stars）

| 属性 | 详情 |
|------|------|
| **作者** | @ivangdavila |
| **功能** | 自我反思 + 自我批评 + 自我学习 + 自组织记忆 |
| **许可** | MIT-0 |

**工作原理**：

与 #1 类似但更进一步，引入了**分层记忆架构**：

```
~/self-improving/
├── memory.md          # HOT：≤100 行，始终加载
├── index.md           # 主题索引
├── projects/          # 按项目的学习
├── domains/           # 按领域（代码、写作、通信）
├── archive/           # COLD：衰减的模式
└── corrections.md     # 最近 50 条纠正记录
```

**三级存储**：

| 层级 | 位置 | 限制 | 加载策略 |
|------|------|------|---------|
| HOT | memory.md | ≤100 行 | 始终加载 |
| WARM | projects/, domains/ | ≤200 行/文件 | 上下文匹配时加载 |
| COLD | archive/ | 无限 | 仅显式查询时加载 |

**自动晋升/降级**：
- 模式在 7 天内使用 3 次 → 晋升到 HOT
- 模式 30 天未使用 → 降级到 WARM
- 模式 90 天未使用 → 归档到 COLD
- 永不删除，只压缩/合并

**自我反思**：完成重要工作后，agent 暂停并评估——结果是否符合预期？什么可以更好？这是模式吗？

**创新点**：分层存储 + 时间衰减 + 自我反思，比简单的日志记录更接近"真正的学习"。

---

### 3. Ontology — 知识图谱（168k 下载，550 stars）

| 属性 | 详情 |
|------|------|
| **作者** | @oswalpalash |
| **功能** | 类型化知识图谱，用于结构化 agent 记忆和可组合技能 |
| **许可** | MIT-0 |

**工作原理**：

将 agent 的记忆从扁平文本升级为**结构化知识图谱**：

```
Entity: { id, type, properties, relations, created, updated }
Relation: { from_id, relation_type, to_id, properties }
```

**核心类型**：Person、Organization、Project、Task、Goal、Event、Document、Message 等

**存储**：基于 JSONL 的 append-only 日志（`memory/ontology/graph.jsonl`）

**约束系统**：通过 `schema.yaml` 定义类型约束，包括必填字段、枚举值、关系基数、无环检查等

**跨 Skill 通信**：其他 skill 可以读写 ontology 图谱。例如邮件 skill 创建一个"承诺"实体，任务 skill 自动拾取并创建对应任务。

**创新点**：将 agent 的知识存储从"笔记本"升级为"数据库"，支持类型检查、关系查询和跨 skill 通信。

---

### 4. Polymarket（126k 下载，81 stars）

| 属性 | 详情 |
|------|------|
| **作者** | @joelchance |
| **功能** | 查询 Polymarket 预测市场的赔率、趋势、事件 |

**工作原理**：让 agent 能够查询预测市场数据——赔率、热门市场、事件搜索、价格走势追踪。纯指令驱动，教 agent 如何调用 Polymarket API。

---

### 5. Multi Search Engine（124k 下载，578 stars）

| 属性 | 详情 |
|------|------|
| **作者** | @gpyangyoujun |
| **功能** | 集成 16 个搜索引擎（7 个国内 + 9 个国际），支持高级搜索 |
| **许可** | MIT-0 |

**工作原理**：

**无需 API Key**，直接用 `web_fetch` 爬取搜索结果页面：
- **国内引擎**：百度、Bing CN、Bing INT、360、搜狗、微信搜索、神马
- **国际引擎**：Google、Google HK、DuckDuckGo、Yahoo、Startpage、Brave、Ecosia、Qwant、WolframAlpha

**智能路由**：自动检测查询语言，中文走国内引擎，非中文走国际引擎

**反爬处理**：内存中动态获取 cookie、403/429 自动重试、速率限制（1-2 秒间隔）

**创新点**：完全免费的多引擎搜索聚合，自动语言路由，无需任何 API key。

---

### 6. PollyReach（95.7k 下载）

| 属性 | 详情 |
|------|------|
| **作者** | @pollyreach |
| **功能** | 给 AI Agent 一个电话号码，能打电话、找联系人、完成电话任务 |

**工作原理**：通过 PollyReach API 让 agent 能真正"打电话"——预约餐厅、联系客服、确认预约等。这是 agent 从数字世界延伸到现实世界的桥梁。

---

### 7. Agent Browser（93.3k 下载，336 stars）

| 属性 | 详情 |
|------|------|
| **作者** | @matrixy |
| **功能** | 无头浏览器自动化，基于可访问性树快照和 ref 元素选择 |

**工作原理**：为 AI agent 优化的无头浏览器自动化 CLI。不依赖截图/视觉，而是通过**可访问性树（Accessibility Tree）** 来理解页面结构，token 消耗远低于截图方案。

---

### 8. Nano Banana Pro（88.7k 下载，348 stars）

| 属性 | 详情 |
|------|------|
| **作者** | @steipete（OpenClaw 创始人） |
| **功能** | 使用 Gemini 3 Pro Image 生成/编辑图片 |

**工作原理**：封装 Gemini 3 Pro 的图像生成能力，支持 text-to-image 和 image-to-image 编辑，支持 1K/2K/4K 分辨率。

---

### 9. Obsidian（84k 下载，340 stars）

| 属性 | 详情 |
|------|------|
| **作者** | @steipete |
| **功能** | 操作 Obsidian 笔记库，通过 obsidian-cli 自动化 |

**工作原理**：让 agent 能读写 Obsidian Vault 中的 Markdown 笔记，实现"第二大脑"与 AI agent 的双向集成。

---

### 10. API Gateway（70.2k 下载，351 stars）

| 属性 | 详情 |
|------|------|
| **作者** | @byungkyu |
| **功能** | 连接 100+ API（Google Workspace、Microsoft 365、GitHub、Notion、Slack 等），托管 OAuth |

**工作原理**：统一的 API 网关，解决 agent 调用第三方 API 时的 OAuth 认证痛点。一次配置，agent 即可访问 100+ 服务。

---

### 其他热门 Skills 一览

| 排名 | Skill | 下载量 | 功能 |
|------|-------|--------|------|
| 11 | **Prismfy Web Search** | 77.3k | 跨 10 个引擎的默认 Web 搜索 |
| 12 | **Word/DOCX** | 63.2k | 创建/编辑 Microsoft Word 文档 |
| 13 | **Mcporter** | 57.4k | MCP 服务器管理和调用 |
| 14 | **Excel/XLSX** | 57k | 创建/编辑 Excel 工作簿 |
| 15 | **IMAP/SMTP Email** | 39.6k | 通过 IMAP/SMTP 读写邮件 |
| 16 | **Powerpoint/PPTX** | 35.7k | 创建/编辑 PowerPoint 演示文稿 |
| 17 | **Skill Finder Cn** | 32.6k | 中文技能查找器，帮助发现和安装 skills |
| 18 | **ClawdHub** | 32k | ClawHub CLI 管理 skills |
| 19 | **Discord** | 30.9k | 控制 Discord 机器人 |
| 20 | **Playwright** | 28.3k | 浏览器自动化（MCP + 爬虫） |
| 21 | **Data Analysis** | 27k | 数据分析和可视化 |
| 22 | **Web Search by Exa** | 26.4k | Exa 神经搜索 |
| 23 | **Baidu Wenku AIPPT** | 25.6k | 百度文库 AI 生成 PPT |
| 24 | **Peekaboo** | 24.7k | macOS UI 截图和自动化 |

---

## 六、Skills 的分类

从 ClawHub 的分类体系来看，skills 可以分为以下几大类：

### 1. 自我改进类（Self-Improvement）
- **self-improving-agent**：错误和纠正日志
- **Self-Improving + Proactive Agent**：分层记忆 + 自我反思
- **ontology**：知识图谱结构化记忆

这类 skill 的核心思想是让 agent 越用越好，实现**记忆持久化和模式学习**。

### 2. 搜索/信息获取类
- **Multi Search Engine**：16 引擎聚合搜索
- **Prismfy Web Search**：10 引擎搜索
- **Web Search by Exa**：神经搜索
- **Answer Overflow**：Discord 社区问答搜索
- **X Search**：Twitter/X 搜索

### 3. 办公文档类
- **Word/DOCX**、**Excel/XLSX**、**Powerpoint/PPTX**：Office 三件套
- **Baidu Wenku AIPPT**：百度文库 AI 生成 PPT
- **Data Analysis**：数据分析和可视化

### 4. 通信平台类
- **Slack**、**Discord**、**Trello**：团队协作工具
- **IMAP/SMTP Email**：邮件收发
- **PollyReach**：电话语音

### 5. 浏览器自动化类
- **Agent Browser**：无头浏览器
- **Playwright**：Playwright MCP 浏览器自动化
- **Peekaboo**：macOS UI 自动化

### 6. API 集成类
- **API Gateway**：100+ API 统一网关
- **Mcporter**：MCP 服务器管理
- **Caldav Calendar**：日历同步

### 7. 媒体生成类
- **Nano Banana Pro**：图像生成/编辑（Gemini）

### 8. 知识管理类
- **Obsidian**：Obsidian 笔记集成

### 9. 金融/市场类
- **Polymarket**：预测市场
- **AdMapix**：广告情报和应用分析

---

## 七、OpenClaw 的 Skills 加载机制（技术细节）

### 加载优先级（从低到高）

```
skills.load.extraDirs  →  bundled skills  →  ~/.openclaw/skills
→  ~/.agents/skills  →  <workspace>/.agents/skills  →  <workspace>/skills
```

**workspace 中的 skill 优先级最高**，可以覆盖任何同名 skill。

### 门控机制（Gating）

Skills 可以通过 metadata 定义加载条件：

```yaml
metadata:
  {
    "openclaw": {
      "requires": {
        "bins": ["uv"],           # 需要 PATH 中有这些可执行文件
        "env": ["GEMINI_API_KEY"], # 需要这些环境变量
        "config": ["browser.enabled"] # 需要配置项为 true
      },
      "primaryEnv": "GEMINI_API_KEY",
      "os": ["darwin", "linux"]    # 限制操作系统
    }
  }
```

### Token 影响

Skills 会注入到系统提示中，成本可计算：

```
total = 195 + Σ (97 + len(name) + len(description) + len(location))
```

粗略估算：~24 tokens/skill + 字段内容，所以 20 个 skill 约消耗 500-800 tokens 系统提示空间。

### 安全机制

- **VirusTotal 扫描**：ClawHub 上的 skill 会经过 VirusTotal 恶意代码扫描
- **OpenClaw 内置扫描**：内置危险代码扫描器，`critical` 发现默认阻止安装
- **沙箱运行**：不信任的 skill 可以在 Docker 沙箱中运行
- **路径限制**：只接受 realpath 在配置根目录内的 skill 文件

---

## 八、如何创建一个 Skill

### 最小示例

```bash
mkdir -p my-skill
cat > my-skill/SKILL.md << 'EOF'
---
name: my-skill
description: Does X when the user asks for Y.
---

# My Skill

## When to use
Use this skill when...

## Steps
1. First, do this...
2. Then, do that...
EOF
```

### 最佳实践

1. **description 要具体**：包含 what + when，让 agent 知道何时激活
2. **SKILL.md 保持在 500 行以内**：详细内容放到 `references/` 中
3. **包含具体示例**：输入/输出示例帮助 agent 理解预期行为
4. **覆盖边界情况**：常见错误和处理方式
5. **name 必须匹配目录名**：`my-skill/SKILL.md` 中 `name: my-skill`
6. **不要在 skill 目录中放 README.md**：会与 SKILL.md 冲突

### 发布到 ClawHub

```bash
clawhub publish ./my-skill
```

---

## 九、Skills 生态的关键趋势

### 1. 自我改进是最大需求

下载量 Top 3 中有 2 个是自我改进类 skill（共 572k+ 下载）。用户最渴望的不是让 agent 多一个功能，而是让 agent **记住教训、越来越好**。

### 2. "无 API Key" 是杀手特性

Multi Search Engine（124k 下载）的成功说明：**免费、无需配置的 skill 最容易传播**。需要 API key 的 skill 有更高的使用门槛。

### 3. 中国社区高度活跃

多个热门 skill 有中文版本或专门面向中国用户（Skill Finder Cn、Multi Search Engine 含 7 个国内搜索引擎、百度文库 AIPPT），反映出中国开发者社区的积极参与。

### 4. 从 "Chatbot" 到 "Digital Employee" 的转变

PollyReach（打电话）、IMAP/SMTP（收发邮件）、Office 三件套等 skill 的流行，说明用户正在将 agent 从聊天机器人推向**真正的数字员工**——能处理现实世界任务。

### 5. 标准化带来互通

AgentSkills 开放标准被 20+ 产品采纳，意味着一个好 skill 可以跨 Claude Code、Copilot、OpenClaw、Hermes 等平台运行。这种**互通性**正在形成网络效应。

### 6. 安全是头等大事

ClawHub 引入 VirusTotal 扫描，OpenClaw 内置危险代码扫描器——skill 本质上是给 agent 的"指令注入"，必须像对待代码依赖一样对待安全。

---

## 十、总结

| 维度 | 现状 |
|------|------|
| **生态规模** | 57,300+ skills，12M+ 下载 |
| **标准化** | AgentSkills 开放标准，20+ 产品采纳 |
| **最大平台** | ClawHub（OpenClaw 生态） |
| **最热品类** | 自我改进 > 搜索 > 办公文档 > 通信 |
| **最大趋势** | 让 agent 学习和记忆（self-improving）|
| **核心优势** | 轻量、纯文本、可审计、跨平台 |
| **主要挑战** | 安全审查、质量参差不齐、标准碎片化 |

Skills 的意义不只是"给 agent 加功能"——它代表了一种新范式：**用声明式的自然语言指令来编程 AI agent 的行为**。这比传统的 API 开发低一个数量级的门槛，任何能写 Markdown 的人都能创建 skill。当 skill 生态成熟到一定程度，AI agent 将变成一个**通用的个人操作系统**，通过 skill 插件覆盖生活和工作的方方面面。
