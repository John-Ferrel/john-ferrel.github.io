---
title: Agent and Skills Development
tags:
  - ai-agent
  - opencode
  - knowledge-base
  - workflow
  - skills
draft: false
created: 2026-06-17 23:54
modified: 2026-06-17 23:54
---
最近我开始整理公司内部的 Agent Workspace，并准备正式开发 Agent 和 Skills。
Agent 和 Skill 是两层不同的抽象。

> **Agent 负责“谁来做事”。**  
> **Skill 负责“某类事情应该怎么做”。**

再加上项目级文档、工具脚本和权限控制，一个比较完整的 Agent 系统应该是这样的：

```text
AGENTS / README / START_HERE 负责项目规则和上下文
Agent 负责角色、职责、边界和权限
Skill 负责某类任务的标准流程
Tool / Script 负责真实执行动作
Permission 负责控制哪些操作允许、询问或禁止
````

## 1. 基本模型

在一个可维护的 Agent Workspace 里，我更倾向于把系统拆成几层：

```text
User Request
  ↓
Primary Agent 理解任务
  ↓
读取 AGENTS.md / README.md / START_HERE.md
  ↓
选择合适的 Skill
  ↓
按 Skill 的 SOP 调用工具、编辑文件或生成交付物
  ↓
输出结果
```

这里的关键是：  
Agent 不应该每次都重新发明工作流程。真正稳定的能力应该沉淀到 Skill 里；真正可执行、可测试、可复用的动作应该沉淀到 script 或 tool 里。

## 2. AGENTS.md：项目级宪法

`AGENTS.md` 是项目里的“宪法”, 需要告诉所有 Agent：
特别的, 也可以给出一些比较抽象的约束, 比如 superpowers的AGENTS.md 中, 写这个项目有95%的概率拒绝pr, 因此必须更加谨慎。

```text
这个项目是什么
当前工作区在哪里
目录结构是什么
哪些文件是权威入口
哪些目录可以读
哪些目录可以写
哪些命令可以执行
什么操作必须先 ask
什么操作禁止执行
交付物应该放在哪里
多人或多个 agent 同时工作时如何避免冲突
```

例如在公司知识库项目里，可以约定：

```text
repo/ 是项目权威目录
docs/ 是正式知识库内容
inbox/ 是原始材料输入区
deliverables/ 是对外或对内交付物输出区
.opencode/agents/ 存放 Agent 定义
.opencode/skills/ 存放 Skill 定义
scripts/ 存放可执行工具脚本
```

这类信息不用每次都在 prompt 里重复，可以固化在AGENTS.md。

## 3. README.md 与 START_HERE.md

三个文件有分工：

```text
AGENTS.md
给 Agent 看的规则文件。

README.md
给人看的项目说明，介绍项目目标、结构和使用方法。

START_HERE.md
给新人或新 Agent 的第一入口，告诉它进入项目后应该先读什么、先检查什么、不要做什么。
```

三者可以有重复，但关注点不同。

`AGENTS.md` 更偏向规则。  
`README.md` 更偏向介绍。  
`START_HERE.md` 更偏向启动流程。

对于 Agent Workspace 来说，`START_HERE.md` 很有价值，因为它能降低新会话、新用户、新 Agent 进入项目时的上下文损失。

## 4. Agent：一个有职责和边界的工种

Agent 可以理解成一个“工种配置”。

```text
Agent = 角色定位 + 工作边界 + 默认行为 + 可用工具 + 权限 + 模型偏好
```

它不是越全能越好。一个好的 Agent 应该知道自己负责什么，也知道自己不应该做什么。

例如公司知识库项目里，可以先设计几个 Agent：

```text
kb-architect
负责知识库架构、目录设计、流程设计和权限边界。

kb-doc-writer
负责把草稿、聊天记录、操作记录整理成正式文档。

kb-ingestion-operator
负责处理 inbox/ 里的上传材料，做初步清洗、规范化和入库准备。

kb-reviewer
负责审查文档一致性、安全边界、RAG 友好性和可读性。
```

这样拆分之后，每个 Agent 的职责会清楚很多。

### 示例：kb-doc-writer

```md
---
description: Writes and maintains company knowledge-base documentation
mode: subagent
temperature: 0.2
permission:
  bash:
    "*": ask
    "git status": allow
    "git diff *": allow
  edit: ask
---

You are the company knowledge-base documentation writer.

Your job:
- Turn rough notes, chat logs, operational records, and architecture discussions into clear Markdown documents.
- Prefer editing files under docs/.
- Do not modify deployment scripts, credentials, Docker configs, or opencode permissions unless explicitly asked.
- When the task involves workspace structure, first read START_HERE.md, README.md, and AGENTS.md.
- When writing user-facing docs, separate beginner instructions from operator notes.
- Always preserve uncertainty instead of inventing missing details.

Output style:
- Use clear headings.
- Prefer operational steps over abstract explanation.
- Include command blocks only when the command is safe and relevant.
```

这个 Agent 的特点是：明确知道自己主要维护 `docs/`，不应该随便动部署脚本和权限配置。

## 5. Skill：某类任务的标准操作手册

Skill 是“某类任务怎么做”的方法包。

如果 Agent 是工种，那么 Skill 就是 SOP。

比如：

```text
kb-sop-writing
把操作记录整理成 SOP。

kb-doc-normalization
把上传资料规范化为知识库 Markdown。

kb-rag-chunking
把长文档拆成适合 RAG 检索的 chunk。

kb-agent-workspace
解释公司 Agent Workspace 的目录结构、权限边界和协作规则。

kb-deliverable-export
规定交付物如何生成、命名和放置。
```

一个好的 Skill 应该包含：

```text
什么时候使用
输入是什么
需要先读哪些文件
输出到哪里
执行步骤是什么
验收标准是什么
常见错误是什么
安全边界是什么
```

### 示例：kb-sop-writing

```md
---
name: kb-sop-writing
description: Convert rough operational notes into a clear company knowledge-base SOP document.
compatibility: opencode
metadata:
  audience: internal-operators
  output: markdown
---

# KB SOP Writing Skill

## When to use

Use this skill when the user asks to turn a deployment record, troubleshooting note, chat summary, or operational workflow into a reusable SOP.

Typical tasks:
- Write setup guide
- Write troubleshooting guide
- Write onboarding guide
- Convert chat notes into docs
- Document agent workspace operations

## Required reading

Before writing:
1. Read `AGENTS.md`.
2. Read `START_HERE.md` if the task involves workspace structure.
3. Search existing `docs/` for related documents to avoid duplication.

## Output location

Prefer:
- `docs/operations/` for operational procedures
- `docs/agent-guides/` for agent/user instructions
- `docs/architecture/` for architecture decisions

## SOP structure

Use this structure:

1. Purpose
2. Audience
3. Prerequisites
4. Directory / system context
5. Procedure
6. Verification
7. Common failures
8. Safety notes
9. Related files

## Rules

- Do not invent commands.
- If a command is inferred but not verified, mark it as "to verify".
- Do not expose tokens, passwords, private URLs, or personal identifiers.
- Prefer copy-pasteable commands.
- Keep beginner-facing explanations separate from operator notes.

## Acceptance checklist

The SOP is complete only if:
- A new user knows where to start.
- An operator knows which directory to use.
- Risky commands are clearly marked.
- Verification steps are included.
- The document has a stable location under `docs/`.
```

这类 Skill 的作用是让 Agent 每次处理同类任务时都有稳定流程，而不是临场发挥。

## 6. Agent 与 Skill 的区别

我现在会这样区分：

```text
这是一个工种吗？
→ 做 Agent。

这是一个可复用流程吗？
→ 做 Skill。

这是所有 Agent 都应该知道的项目背景吗？
→ 写进 AGENTS.md / README.md / START_HERE.md。

这是一个可执行动作吗？
→ 写成 script / tool。

这是一次性任务说明吗？
→ 保留在当前 session，不要过度固化。
```

举个例子：

```text
用户说：帮我把这批上传材料整理成知识库文档。

Agent：
kb-ingestion-operator
负责处理 inbox/ 里的资料。

Skill：
kb-doc-normalization
规定如何识别材料、转 Markdown、命名、保留来源和标注不确定内容。

Tool：
scripts/normalize_docs.py
真正批量转换和检查文件。

Permission：
允许读 inbox/，允许写 docs/staging/，批量删除必须 ask。
```

这个例子里，每一层都有自己的职责。

如果所有事情都写进一个巨大 prompt，最后会变得不可维护。  
如果所有事情都拆成 Agent，又会导致 Agent 太碎、难以调度。  
如果所有事情都写成脚本，又会失去 LLM 对上下文和模糊任务的处理能力。
(其实感觉和 SillyTavern的prompt 编排有点类似)

比较合理的做法是：  
**项目规则固化到文档，角色固化到 Agent，流程固化到 Skill，动作固化到工具。**

## 7. 推荐目录结构

对于公司知识库项目，我比较推荐这样的结构：

```text
/srv/company-agent/projects/company-kb/repo/
├── AGENTS.md
├── README.md
├── START_HERE.md
├── opencode.jsonc
├── docs/
│   ├── architecture/
│   ├── operations/
│   ├── workflows/
│   ├── agent-guides/
│   └── kb-standards/
├── inbox/
│   ├── raw/
│   ├── uploads/
│   └── staging/
├── deliverables/
│   ├── reports/
│   ├── exports/
│   └── user-facing/
├── scripts/
│   ├── normalize_docs.py
│   ├── chunk_markdown.py
│   └── validate_kb.py
└── .opencode/
    ├── agents/
    │   ├── kb-architect.md
    │   ├── kb-doc-writer.md
    │   ├── kb-ingestion-operator.md
    │   └── kb-reviewer.md
    └── skills/
        ├── kb-agent-workspace/
        │   └── SKILL.md
        ├── kb-doc-normalization/
        │   └── SKILL.md
        ├── kb-rag-chunking/
        │   └── SKILL.md
        ├── kb-sop-writing/
        │   └── SKILL.md
        └── kb-deliverable-export/
            └── SKILL.md
```

这里最重要的是边界清楚：

```text
docs/ 是正式内容
inbox/ 是输入材料
deliverables/ 是交付结果
scripts/ 是可执行工具
.opencode/agents/ 是角色定义
.opencode/skills/ 是能力定义
```

## 8. 不要一开始过度设计

Agent 和 Skill 的设计很容易过度抽象。

我觉得第一版不需要很多内容。最小可用版本可以是：

```text
AGENTS.md
README.md
START_HERE.md

.opencode/agents/
  kb-doc-writer.md
  kb-reviewer.md

.opencode/skills/
  kb-agent-workspace/SKILL.md
  kb-sop-writing/SKILL.md
  kb-doc-normalization/SKILL.md
```

这已经足够开始处理真实任务了。

后续可以根据真实使用情况逐步增加：

```text
kb-ingestion-operator
kb-rag-chunking
kb-deliverable-export
kb-release-note
kb-security-review
```

关键是**真实任务打磨 Agent 体系。**

## 9. 我的开发顺序

我接下来会按这个顺序推进：

```text
1. 先写稳 AGENTS.md / README.md / START_HERE.md
2. 做第一个通用 Skill：kb-sop-writing
3. 做一个只读审查 Agent：kb-reviewer
4. 做一个文档写作 Agent：kb-doc-writer
5. 用真实任务测试，比如整理 cc-connect 新用户接入 SOP
6. 再抽象出文档规范化、RAG chunk、交付物导出等 Skills
```

这样做的好处是，每一步都能产生真实价值, 在实际工作流中逐步沉淀能力。

## 10. 总结

Agent 和 Skills 开发的看起来像是更复杂的 prompt，实际上是把工作流拆成稳定的层次：

```text
项目规则：AGENTS.md / README.md / START_HERE.md
角色分工：Agent
任务流程：Skill
执行动作：Tool / Script
安全边界：Permission
```

我目前对它的理解可以概括成一句话：

> **Agent 让 AI 知道自己是谁，Skill 让 AI 知道这类事情应该怎么做。**

如果这两层设计清楚，后面的多人协作、多入口接入、知识库维护、文档处理和 RAG 工作流，都会更容易沉淀成可复用系统。
