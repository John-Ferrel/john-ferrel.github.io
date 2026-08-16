---
title: AI Coding in a Tiny Team
draft: false
tags:
  - ai
  - agent
  - coding
  - git
  - opencode
  - cc-connect
  - team-workflow
created: 2026-08-13 23:27
modified: 2026-08-14 00:01
---
最近，我们在三个人的开发团队里调整了几处 coding workflow。

团队共用一台开发服务器。每个人有自己的 Linux user，也各自维护一份 repo clone。团队共享数据可以由成员共同访问；个人 workspace、未提交代码和 credential 仍然保留在各自的 user space 里。

下文把 workspace 作为工作目录的统称：个人 workspace 是开发者自己的 repo clone，qbot workspace 是单独 clone 出来的目录；`worktree` 指实际被修改的那份工作目录。`user space` 则指各自 Linux user 拥有的个人空间。

在此基础上，我们又接入了一个通过 IM 使用的 coding agent。

现在的结构是：

```text
Developer A ─→ own workspace
Developer B ─→ own workspace
Developer C ─→ own workspace

Feishu
  ↓
cc-connect
  ↓
OpenCode qbot
  ↓
dedicated workspace
```

qbot 不属于任何一个人的日常开发环境，是团队共享的 coding agent。

这套结构把三件事分开：哪个 workspace 可以动、Agent 能做什么、结果如何回到 Git workflow。

## 给 Agent 单独准备一个 workspace

起初，我们直接让 IM bot 使用某个开发者已经存在的 repo。

后来，我们给它额外 clone 了一份。

真正麻烦的地方在 worktree 状态。

开发者自己的 repo 里可能同时存在：

- 当前正在开发的 branch；
- 没有 commit 的修改；
- 临时实验；
- 本地配置；
- 一个尚未结束的 Agent session。

如果 IM bot 直接在这里工作，每次操作前都要先判断哪些文件可以动、branch 能不能切，以及现有修改属于谁。

qbot 的执行会依赖某个开发者当前的工作状态。异步任务启动前，worktree 越复杂，需要判断的内容越多。

单独 clone 后，qbot 只需要面对自己的 worktree：

```text
/home/a/projects/repo
/home/b/projects/repo
/home/c/projects/repo

/home/x/projects/repo-qbot
```

这个 worktree 只属于 qbot。

每次开始写任务前，先确认 worktree clean，并从 `origin/main` 更新。开始修改后，新建：

```text
qbot/<topic>
```

分支。

完成后依次跑测试、commit、push，再创建 PR。

遇到未知的 dirty state、rebase conflict、测试失败或者 push rejected 时，就停下来报告；这些状态不由它自行恢复。

流程沿用开发者的 workflow，执行者换成了 Agent。

## IM 入口需要更高的 Agent 自主权

qbot 使用专门的 OpenCode agent。

它的权限比普通交互式 Agent 更宽。

正常的 repository work，例如：

- 读取和修改代码；
- 创建、删除项目内文件；
- 运行 shell command；
- 使用 `uv` 修改依赖；
- 运行有限范围的测试；
- commit；
- push task branch；
- 创建或更新 PR；

这些操作通常都可以自行完成。

风险更高的操作仍然禁止或需要停下来确认：

- 不 force push；
- 不 rewrite 已发布 commit；
- 不暴露或复制 credential；
- 不自行处理未知的 dirty worktree；
- 不启动明显重型的数据、训练或 backtest 任务；
- destructive operation 不在默认允许范围内；
- 默认不直接修改 `main`。

qbot 的主要入口是 IM，确认次数会直接变成消息往返。

如果每一个正常步骤都需要：

```text
Can I edit this file?
Can I run this command?
Can I commit?
Can I push?
```

在 IM 里，这种交互很快会变得笨重。

权限按风险分成三类：

```text
normal in-scope work
    → allow

missing important decision
    → ask

high-risk operation
    → deny / stop
```

`ask` 只用于补足重要决策，不承担主要的权限控制。

在 terminal 里，频繁确认的成本较低；换成 IM 后，每次确认都会增加一次消息往返。

配置的重点是先划定 Agent 可以活动的范围，再让它在范围内完成任务。

## Agent 仍然通过 Git 与其他人协作

qbot 的执行权限比较宽，协作流程仍沿用现有的 Git workflow。

正常写任务仍然是：

```text
request
  ↓
qbot/<topic>
  ↓
implementation
  ↓
tests
  ↓
commit
  ↓
push
  ↓
pull request
```

`main` 默认保持保护状态。

Agent workspace 和个人 workspace 各自独立，大家也不需要直接读取或修改 qbot 的 worktree。

需要共享的结果最终通过 Git 交付。

在团队协作里，Agent 承担的是 contributor 的角色：独立创建 branch 和 PR，结果通过 Git 回到团队流程。

## 在 PR 上增加独立的 review

我们还在 GitHub 上接入了 OpenCode App。

review 改为按需触发：

```text
/oc
```

一个 PR 在开发过程中可能连续 push 很多次，中间状态未必值得重新做完整 review。有时只是刚修复了一个小问题，或者 PR 还没有准备好接受 review。

什么时候触发 review，仍然由人决定。

流程如下：

```text
human / qbot
     ↓
 implementation
     ↓
     PR
     ↓
   /oc
     ↓
OpenCode review
     ↓
human decision
     ↓
   merge
```

review agent 和负责实现的 Agent 也不共享工作上下文。

review agent 看到的是 PR 和代码变化，不会看到实现过程中累积的整段 conversation。

它负责独立检查 PR，不承接原实现 Agent 的后续工作。

## 一个 shared bot 带来的问题

目前 qbot 只有一个 workspace。

三个人的团队暂时可以靠一个简单约定维持：

> 同一时间只允许一个 write-producing task 使用 qbot workspace。

其他 conversation 可以查询、解释或 review，但不能同时修改这个 worktree。

这个限制已经写进了 qbot 的 agent instruction：

```text
Keep one write-producing task active in the qbot clone.
Concurrent conversations may inspect or review,
but must not mutate the same worktree.
```

团队只有三个人；每个人也有自己的 user space 和 coding agent，很多工作可以直接在个人环境里完成。这个限制还没有造成明显问题。

使用频率上升后，这个结构的边界会变得明显。

当前关系是：

```text
Developer A ─┐
Developer B ─┼─→ qbot → one workspace
Developer C ─┘
```

查询和讨论可以并发；同一个 worktree 上的写操作不能安全并发。

最直接的处理方式是给每个人准备一个 bot 和一个独立 workspace：

```text
Developer A → qbot-a → workspace-a
Developer B → qbot-b → workspace-b
Developer C → qbot-c → workspace-c
```

问题会从“接一个 IM bot”扩展到任务调度和执行环境管理：

- bot 应该怎么路由；
- task 和 workspace 如何绑定；
- 一个用户能不能同时启动多个任务；
- workspace 是长期存在还是按 task 创建；
- conversation 和执行环境是什么关系；
- 谁负责回收 branch 和 worktree。

cc-connect 作为 communication bridge 足够简单，适合从单个 bot 开始。

当需求从：

```text
IM → one coding agent
```

扩展到：

```text
users
  ↓
task routing
  ↓
multiple agent workers
  ↓
isolated workspaces
```

这时，系统的重点也从消息转发转向任务调度和环境隔离。

## 目前的结构

当前结构如下：

```text
shared Git repository
├── Developer A → personal workspace
├── Developer B → personal workspace
├── Developer C → personal workspace
└── qbot/<topic> → PR → optional /oc → merge

Feishu → cc-connect → qbot → dedicated clone
```

服务器、共享数据、Git repository 和 qbot 可以共用；个人 workspace、Agent worktree、task branch 和 credentials 分开管理。mutable state 不跨越这些边界。

当前保留的边界很简单：

```text
personal workspace   → separate
credentials          → separate
agent worktree       → separate
task branch          → separate
main                 → protected
merge                → explicit
```

在这些边界内，Agent 可以相对自由地完成一次正常的 coding task。

接下来要验证两件事：多个 Agent workspace 如何管理，以及 task、conversation 和 workspace 如何关联。先把这几个关系厘清，再决定是否继续增加 permission rule。
