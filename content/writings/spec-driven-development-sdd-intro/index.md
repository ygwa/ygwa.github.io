---
title: "SDD 工具实战：Spec Kit、OpenSpec、Kiro 使用指南"
date: 2026-02-26T19:47:00+08:00
draft: false
tags:
- 软件开发
- AI开发
- 工具使用
- 开发模式
---

## 为什么我们需要 SDD（规范驱动开发）？

### 痛点：AI 时代的「Vibe Coding」陷阱

随着 Cursor、GitHub Copilot、Claude 等 AI 编码助手的普及，软件开发的门槛被大幅拉低。我们进入了一个依赖直觉和对话的阶段——「Vibe Coding」（氛围编程）。开发者只需敲下几行提示词：“帮我写个登录页面，带验证码，UI 现代一点”，代码就会自动生成。

在项目初期，这种方式极具爆发力。但当项目体积逐渐膨胀、逻辑变得复杂时，「Vibe Coding」的致命缺陷就会暴露无遗：

1. **意图漂移（Prompt Drift）**：AI 的上下文窗口是有限的。当你通过几十轮对话不断打补丁时，AI 早就忘记了最初的架构设计，开始胡编乱造或引入冗余逻辑。
2. **打地鼠式 Debug**：修好 A 模块的 Bug，却搞坏了 B 模块的功能。因为没有全局规范的约束，AI 每次生成的代码都在破坏原有的系统内聚性。
3. **架构的“私有化”**：项目的核心业务逻辑和架构决策，全都散落在开发者与 AI 的历史对话框里。团队里的其他人根本无法接手，一旦关闭了那个 Chat 窗口，项目就成了一座“黑盒”。

### 拐点：什么时候必须引入 SDD？

当你或你的团队开始出现以下情况时，就是引入 SDD 的最佳时机：

* **提示词越来越长，但 AI 依然听不懂**：你发现自己花在微调 Prompt 上的时间，甚至超过了手写代码的时间。
* **重构变成了灾难**：想要重构一个核心组件，但 AI 助手完全无法理解各个模块之间的依赖关系，频繁给出破坏性的建议。
* **多人协作频频冲突**：两个开发者各自用 AI 生成了不同风格的代码，合并 PR 时发现底层逻辑完全不兼容。

### 目标：SDD 希望解决什么问题？

Spec-Driven Development（SDD，规范驱动开发）的出现，就是为了给狂奔的 AI 加上“方向盘”。它的核心理念是：**先写规范（Spec），再写代码。**

通过 SDD，我们希望达到以下目标：

* **建立人机共识（Single Source of Truth）**：用机器和人都能看懂的结构化 Markdown 文档作为唯一真实的依据。AI 生成代码不再依赖零散的对话，而是严格锚定这份规范文档。
* **将思考与执行解耦**：让人类开发者回归到「系统设计」和「需求定义」的高价值工作中，把机械的「代码实现」和「测试生成」完全交给 AI。
* **确保长期可维护性**：不管换多少个 AI 模型，不管过多久，只要规范文档（Spec）还在，随时可以重新生成或安全地重构整个项目。

简而言之，**SDD 让 AI 从一个“聪明的盲人摸象者”，变成了一个“有图纸的建筑工人”。**

本文将详细介绍三款主流 SDD 工具的使用方法和标准工作流。

## Spec Kit（GitHub）

### 工具定位

[Spec Kit](https://speckit.org/) 是 GitHub 推出的开源 SDD 工具包，与 GitHub Copilot 等生态深度融合，目前拥有 28K+ GitHub Stars。它通过斜杠命令与 AI 代理交互，执行严格的**四阶段工作流**，适合在现有编辑器（VSCode、JetBrains 等）中集成使用。

### 安装与初始化

```bash
# 使用 uv 安装（推荐）
uv tool install specify-cli --from git+https://github.com/github/spec-kit.git

# 初始化新项目
specify init my-project --ai copilot

# 或在现有项目中初始化
specify init . --ai copilot

```

### 标准 SDD 结构

```text
.specify/
├── memory/
│   ├── constitution.md        # 项目级原则和规范
│   ├── context.md             # 当前会话上下文
│   └── history.md             # 交互历史
├── specs/                     # 功能规范文档
│   └── feature-name/
│       ├── spec.md            # 需求规范
│       ├── plan.md            # 技术规划
│       └── tasks.md           # 任务拆解清单
└── prompts/                   # 可复用的提示词模板

```

### 开发流程

Spec Kit 拒绝从需求直接跳到代码，而是强制执行规划与拆解：

**第一步：建立项目宪法**
使用 `/speckit.constitution` 创建全局原则（如代码质量、测试要求等），生成 `.specify/memory/constitution.md`，作为后续所有 AI 交互的基准。

**第二步：编写规范（Specify）**

```text
/speckit.specify 构建一个任务管理应用，具备创建、分类、优先级和提醒功能。

```

生成 `spec.md`，明确「要做什么（What）」和「为什么做（Why）」。

**第三步：技术规划（Plan）**
执行 `/speckit.plan`。AI 会根据规范生成系统架构设计、组件拆解和技术方案，输出 `plan.md`。

**第四步：拆解任务（Tasks）**
执行 `/speckit.tasks`。将技术方案转化为具体的、可执行的检查清单，输出 `tasks.md`。

**第五步：实施代码（Implement）**
执行 `/speckit.implement`。AI 严格按照 `tasks.md` 中的清单，逐步生成代码并打钩。

---

## OpenSpec

### 工具定位

[OpenSpec](https://openspec.dev/) 强调维护单一、统一的规范文档作为系统设计的权威参考，目前拥有 25K+ GitHub Stars。它支持 20+ 主流 AI 编码工具（Cursor、Claude Code 等），采用「意图驱动」的工作流。

### 安装与初始化

```bash
# 使用 npm 安装
npm install -g @fission-ai/openspec@latest

# 初始化项目
cd your-project
openspec init

```

### 标准 SDD 结构

OpenSpec 的核心是统一规范结构，包含三种工件类型：

**1. 变更规范（Delta Specs）**：提议的修改，如 `changes/add-dark-mode/` 下的 `proposal.md`、`specs/`、`design.md` 和 `tasks.md`。
**2. 真实来源规范（Source of Truth Spec）**：系统当前状态的权威规范，存放在 `specs/` 目录下（如 `domains/`、`apis/`）。
**3. 归档规范（Archived Specs）**：保留历史变更记录，存放在 `archive/` 目录下。

### 开发流程

1. **创建新需求**：使用 `/opsx:new "添加深色模式功能"` 发起提案。
2. **生成工件**：OpenSpec 依次生成 `proposal.md`（提案）、`specs.md`（详细需求）、`design.md`（技术方案）和 `tasks.md`（实现清单）。
3. **审核与调整**：人工审核并按需修改 Markdown 文档。
4. **执行任务**：输入 `/opsx:apply`，AI 按清单逐项实现。
5. **归档**：变更合并后，使用 `/opsx:archive` 将变更规范归档到真实来源规范中。

---

## Kiro（AWS）

### 工具定位

[Kiro](https://kiro.dev/) 是由 AWS 团队主导推出的首款集成 SDD 理念的 AI IDE。它将规范作为「执行规范」，并使用**属性测试（Property-Based Testing）**验证代码的绝对正确性，非常适合高复杂度和高可靠性要求的工程。

### 安装

Kiro 是桌面应用，下载安装地址：[https://kiro.dev/](https://kiro.dev/)

### 标准 SDD 结构

```text
project/
├── specs/                    
│   ├── requirements.md     # 包含 EARS 语法的需求文档
│   ├── design.md           # 系统设计
│   └── tasks.md            # 执行清单
├── src/                    # 源代码
├── tests/                  # 包含自动生成的属性测试
└── kiro.yaml               # 项目配置

```

### 开发流程与核心机制

**第一步：使用 EARS 语法描述需求**
在 `requirements.md` 中，使用 EARS（Easy Approach to Requirements Syntax）语法编写自然语言需求。例如：

> "WHEN a user adds a new task to a list, THEN the list's length SHALL increase by 1 AND the task ID SHALL be unique."

**第二步：生成架构与任务**
Kiro 解析需求文档，自动生成 `design.md` 和 `tasks.md`。

**第三步：自动提取并生成属性测试**
这是 Kiro 最强大的特性。它**不需要你手动编写测试代码**，而是直接从 EARS 语法的 Markdown 句子中提取出数学属性，并结合底层框架（如 Python 的 Hypothesis 或 TS 的 fast-check）生成海量的随机边界输入。

**第四步：生成代码并持续验证**
Kiro 边写代码边跑属性测试。如果代码无法通过诸如「优先级始终在有效范围内」这种通用规则的随机轰炸，AI 会自我修正，直到满足规范定义的所有属性。

---

## 工具对比与选择

| 特性 | Spec Kit (GitHub) | OpenSpec | Kiro (AWS) |
| --- | --- | --- | --- |
| 定位 | 编辑器插件 | 统一规范框架 | AI IDE |
| 工作流流派 | 强制四阶段 Prompt 链 | 意图与工件驱动 | EARS 语法驱动 |
| 规范保存 | 锚定式分离结构 | 统一单一真实来源 (SoT) | 执行规范文档 |
| 测试方式 | 传统单元/集成测试 | 传统测试 | 属性测试 (PBT) |
| 适用场景 | 现有 GitHub Copilot 生态 | 大型团队、复杂需求管理 | 高可靠性、零容错工程 |

**如何选择：**

* **已有固定编辑器和生态**：选择 Spec Kit，它的四步法能立竿见影地提升日常 PR 质量。
* **团队协作与大型项目**：OpenSpec 的「单一真实来源规范」能有效解决团队成员间的信息差。
* **追求极致正确性与全新体验**：Kiro 将需求文档直接转化为海量边界测试，是消灭「vibe coding」副作用的终极形态。