---
title: Conversation First or Workspace First
created: 2026-08-11 00:00
modified: 2026-08-13 22:26
tags:
  - ai
  - agent
  - opencode
  - cc-connect
  - astrbot
  - openclaw
  - im
---

最近一直在使用 cc-connect，把 OpenCode 接到飞书等 IM 平台。

最初的问题很简单：

> 除了 cc-connect，还有没有更合适的 Agent Bridge？

我的场景主要有两个：

1. AI Coding
2. 企业自部署的第三方 Agent 应用

一开始，我是在比较框架；后来发现，更重要的是先明确自己想解决什么问题。

## Workspace 是工作的中心

我更希望 Agent Runtime 绑定一个 workspace，而且这个 workspace 本身就是一个 Git Repo。

例如：

```text
merchandise-planning/
├── AGENTS.md
├── skills/
├── tools/
├── knowledge/
├── docs/
├── deliverables/
└── .git/
```

这个 repo 决定了 Agent 的工作范围、工具、规则和产物。

可以把两者简单类比为：

> Workspace 是职位 / 工位，Agent Runtime 是坐在这里工作的员工。

于是：

```text
crypto-research/          → Quant Research Agent
merchandise-planning/     → Merchandise Planning Agent
company-kb/               → Knowledge Agent
```

IM、Terminal、Web 都只是入口：

```text
Terminal ─┐
Feishu ───┼──→ Agent Runtime → Workspace
Web ──────┘
```

这也是 cc-connect 吸引我的地方：它主要负责连接，真正执行工作的还是 OpenCode、Codex、Claude Code 这类 Coding Agent。

## 问题出在群聊

私聊里：

```text
User message ≈ Prompt
```

通常没什么问题。

群聊则不同。

例如：

```text
A: 上海库存最近很低
B: 昨天有促销
C: 调拨还没到

A: @Agent 帮忙看看
```

如果 Bridge 只把包含 `@Agent` 的那条消息交给 Runtime，前三句话就会丢失。

`reply_all` 也无法完整解决这个问题，因为：

```text
Bot 看到消息
```

和：

```text
Bot 需要回复消息
```

属于两种不同的判断。

更合适的做法是把消息分成：

```text
ignore
observe
trigger
```

普通群消息进入一个短期 conversation buffer，不调用 Agent。

真正触发时，再由 Gateway 组装当前工作语境：

```text
上一轮回复后，群里有这些消息：

A: 上海库存最近很低
B: 昨天有促销
C: 调拨还没到

此时 A @了你：

A: 帮忙看看

请结合上述上下文处理当前请求。
```

这段内容属于一次 invocation 的附加上下文，不进入 Agent Memory。

## Session 和 Workspace 分开

同一个 workspace 可以对应多个 session：

```text
Workspace
├── 飞书私聊 → Session A
├── 群聊     → Session B
├── Thread   → Session C
└── Web      → Session D
```

它们不需要共享聊天历史。

需要长期保留的内容应进入 workspace：

```text
docs/
deliverables/
src/
.git/
```

更准确地说：

> Files are the durable truth.

Runtime Session 负责当前 conversation 的工作上下文。

Gateway 只负责当前 IM 场景。

这种划分也能避免额外设计复杂的跨 session memory。

### `/new` 和 Thread

`/new` 应该真正创建一个新的 Runtime Session，旧 session 保留，之后可以通过 `/switch` 切回去。

Thread isolation 遵循同样的思路：

```text
主群   → Session A
Thread → Session B
```

Session B 第一次启动时，可以带入一部分主群近期聊天作为 bootstrap context，后续独立发展。

Gateway 只需要提供简单、可配置的 history limit，例如 `max_messages` / `max_tokens`；长期 compaction 仍由 Agent Runtime 自己处理。

## Bot 和 Workspace

我最开始也考虑过一个 Bot 内部绑定多个 workspace。

但在飞书这类平台上，我更倾向于为不同 workspace 建立独立的 Bot/App：

```text
商品计划 Bot → merchandise-planning/
技术 Bot     → engineering/
知识库 Bot   → company-kb/
```

这样，Bot 身份本身就承担了一部分 routing 和权限语义。

开发环境里，让不同 workspace 使用不同的 session / cwd 就够了。

正式部署时，再通过 Docker、Sandbox、OS User 等方式进行安全隔离。这些属于部署层面的问题，不必混入 conversation 设计。

## 再看几个框架

有了这个区分，再比较 cc-connect、OpenClaw、AstrBot，差异就清楚很多。

### cc-connect

cc-connect 的核心模型与这个思路比较接近：

```text
Workspace
 ↓
External Agent Runtime
 ↓
Runtime Session
```

目前的短板主要在于 Conversation Layer 太薄。

当前更接近：

```text
message
 ↓
trigger?
 ↓
Agent
```

我需要的流程是：

```text
message
 ↓
conversation buffer
 ↓
trigger?
 ↓
context assembly
 ↓
Agent
```

因此，如果继续使用 cc-connect，方向就很明确：保留它的 Project / Runtime / Session 模型，重做 conversation management。

### OpenClaw

OpenClaw 在 conversation 管理上明显成熟得多。

它已经支持类似 pending group history、thread scope、parent context 等机制，和上面的模型很接近。

OpenClaw 的职责也不止 Gateway。

它还包含：

```text
Agent Runtime
Workspace
Session
Memory
Tools
Skills
TUI
```

如果再接入 OpenCode，需要先明确：

```text
谁做决策？
谁管 Session？
谁管 Permission？
谁拥有 Workspace？
```

另一种路径是直接把 OpenClaw 作为主要 Runtime，此时讨论重点就会转向 OpenClaw 与 OpenCode 的组合。

### AstrBot

AstrBot 更强调 conversation、chatbot 和 application runtime。

乍看之下，这与 workspace-first 的思路存在冲突。

不过，它现在提供了独立的 Agent Runner，可以把 Dify、DeerFlow 这类完整 Agent Runtime 放在后面。

理论上也可以这样连接：

```text
Feishu
 ↓
AstrBot
 ↓
OpenCode Runner
 ↓
OpenCode
 ↓
Workspace
```

这时 AstrBot 负责：

```text
IM
conversation
commands
attachments
context assembly
```

OpenCode 负责：

```text
task planning
runtime session
files / shell / tools
workspace
execution permission
```

因此，关键问题在于：

> 为了让 OpenCode 成为真正的 Runtime，需要绕开多少 AstrBot 自己的 Agent / Conversation 机制？

如果只需要一个 OpenCode Runner，并做少量 conversation 调整，它可能很合适。

如果还需要重写 history、session、permission、streaming、tool events，改造成本未必低于修改 cc-connect。

### LangBot 和 NoneBot

LangBot 更像：

```text
IM → Pipeline → AI Backend
```

它比较适合连接 Dify、n8n 一类系统。在 Git Workspace + Coding Runtime 这类模型下，目前没有看到特别明显的优势。

NoneBot 更薄，只负责把不同平台消息抽象成 Event。

它足够干净，但后面的 conversation、session、streaming、permission、coding events 基本都要自己实现，最后很容易走向重新实现一个 cc-connect。

## Chatbot 和 Coding Bot

到这里，几个框架背后的差异可以归结为两种起点。

Chatbot 更自然的起点是：

```text
Conversation First

Conversation
 ↓
Agent
 ↓
Workspace / Tools / Knowledge
```

它首先关心：

* 谁在说话
* 谁在回复谁
* 前面聊了什么
* 当前群聊是什么语境

Coding Agent 更自然的起点是：

```text
Workspace First

Conversation
 ↓
Work Context
 ↓
Agent
 ↓
Workspace
```

它首先关心：

* 当前 repo
* 任务是什么
* 有哪些文件和工具
* 当前状态是什么
* 怎么完成并验证任务

在单用户场景下，两者差别很小。

到了多用户场景，差异才会明显。

Coding Agent 本身不会自动理解多人 conversation，需要先由其他层把 conversation 转换成清晰、可执行的工作语境。

这正是当前需要解决的层次。

最终选择 fork cc-connect、采用 AstrBot + OpenCode Runner，还是使用 OpenClaw，仍然需要在同一个场景下实际运行后再判断。
