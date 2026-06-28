---
title: Building KnowledgeBase with OpenCode 3 - CC Connect
tags:
  - opencode
  - cc-connect
  - agent
  - knowledge-base
  - feishu
  - wechat
  - ai-coding
draft: false
created: 2026-06-14 22:37
modified: 2026-06-16 18:59
---
前两篇[[Building KnowledgeBase with Opencode]]和[[Building KnowledgeBase with OpenCode 2 - Docker and SFTPGo]]主要讨论了如何把 OpenCode 用作知识库工程化工作台：通过目录结构、权限配置、文档约束和工作流，把一个“文档项目”变成 Agent 可读、可改、可审查的 KnowledgeBase。

这一篇稍微往外扩一步：**如果我们不只想在终端或 Web UI 里使用 OpenCode，而是希望它能通过微信、飞书等协作平台被调用.**

因为是周末的探索工作, 这次实践是在个人服务器, 目标是在不引入过重 Docker 架构的情况下，部署一套轻量的结构：

```text
微信 / 飞书
  ↓
cc-connect
  ↓
OpenCode
  ↓
个人 / 团队工作区
  ↓
LLM Provider
```

---

## 1. 为什么需要把 OpenCode 接入微信 / 飞书？

OpenCode 原本更像一个开发者工作台：

```text
SSH / Terminal / TUI / IDE
  ↓
OpenCode
  ↓
项目目录
```

这种方式适合深度开发、调试、查看 diff、运行测试。但它不适合所有场景。

在公司知识库或内部 Agent 场景中，很多用户并不是开发者，他们更习惯：

- 在飞书群里提问；
- 在微信里远程派任务；
- 上传一个文件，让 Agent 整理；
- 让 Agent 检查知识库目录；
- 让 Agent 生成一份交付文档；
- 在群里讨论后让 Agent 执行明确任务。

这就需要一个“协作平台到 Agent Runtime 的Bridge”。

cc-connect 扮演的正是这个角色：

```text
Chat Platform Gateway
  ↓
Session Router
  ↓
Agent Adapter
  ↓
OpenCode Runtime
```

它本身不是模型，也不是知识库系统，而是把微信、飞书、Telegram、Slack 等平台的消息转发给本地或服务器上的 coding agent。

---

## 2. 基础架构

最终结构可以理解为：

```text
用户
  ↓
微信 / 飞书 / 群聊
  ↓
cc-connect
  ↓
OpenCode
  ↓
workspace
  ↓
AGENTS.md / Skills / opencode.jsonc
  ↓
LLM Provider
```

其中每一层职责不同。

### 2.1 微信 / 飞书：用户入口

微信和飞书负责：

- 接收用户消息；
- 接收附件；
- 展示 Agent 回复；
- 在群聊中通过 @bot 触发任务；
- 作为轻量远程控制入口。


### 2.2 cc-connect：消息桥与会话路由

cc-connect 负责：

- 连接不同 IM 平台；
- 判断消息来自哪个用户、哪个平台、哪个群；
- 根据配置找到对应 project；
- 调用 OpenCode；
- 把结果发回聊天平台；
- 维护会话关系。

它可以一个进程管理多个项目：

```toml
[[projects]]
name = "personal-sandbox"

[projects.agent]
type = "opencode"

[projects.agent.options]
work_dir = "/srv/personal-agent/projects/sandbox"

[[projects.platforms]]
type = "weixin"

[[projects.platforms]]
type = "feishu"
```

这意味着一个 `personal-sandbox` 可以同时接微信和飞书，也可以再增加更多平台。

### 2.3 OpenCode：真正的 Agent Runtime

OpenCode 负责：

- 读取工作区；
- 理解 `AGENTS.md`；
- 加载 Skills；
- 调用模型；
- 读写文件；
- 调用 shell；
- 执行开发、文档、分析任务。

模型选择、工具权限和工作区规则不应该主要写在 cc-connect 里，而应该由 OpenCode 项目配置管理。

### 2.4 workspace：真实工作环境

工作区是 Agent 的操作对象，例如：

```text
/srv/personal-agent/projects/sandbox/
├── AGENTS.md
├── opencode.jsonc
├── inbox/
├── work/
├── notes/
├── deliverables/
├── archive/
├── tmp/
└── .opencode/
    └── skills/
```

对公司知识库来说，这个 workspace 可以换成：

```text
/srv/company-kb/
├── AGENTS.md
├── README.md
├── START_HERE.md
├── docs/
├── inbox/
├── deliverables/
└── .opencode/
```

也就是说，个人部署和公司知识库部署的核心差别不在链路(链路是一样的)，而是工作区内容、权限边界和组织规范。

---

## 3. 服务器资源与部署策略

这次实践使用的是一台轻量个人服务器：

```text
CPU: 4 核
内存: 4GB
系统盘: 40GB SSD
带宽: 3Mbps
系统: Ubuntu Server 24.04 LTS
```

这个配置不适合本地跑大模型，但足够运行：

- cc-connect；
- OpenCode；
- 微信 / 飞书连接；
- 少量个人工作区；
- 远程 LLM API 调用。

因此，不推荐在第一版引入 Docker。

更合适的方式是：

```text
一个 Linux 用户 agent
  ↓
一个 cc-connect 进程
  ↓
一个或多个 projects
  ↓
每个 project 一个 work_dir
```

这种方案的优点是：

- 部署简单；
- 占用资源低；
- 日志和配置容易排查；
- 对个人或小团队可信用户足够实用。

缺点是：

- 不是强隔离；
- 多个项目仍可能以同一个 Linux 用户运行；
- 如果允许任意 shell，理论上可以访问该用户有权限访问的其他目录；
- 需要通过权限配置、工作区规则和用户信任来约束。

这类模式更适合：

```text
可信用户 + 独立目录 + 明确权限 + Git 可恢复 + Skills 审查
```

不适合直接暴露给陌生用户或不可信外部用户。

---

## 4. 模型配置：让 OpenCode 管模型，不让 cc-connect 管模型

部署过程中遇到过一个小问题, 直接用 cc cennect的set up：

OpenCode 项目里的 `opencode.jsonc` 已经配置了模型：

```jsonc
{
  "model": "opencode-go/deepseek-v4-flash",
  "small_model": "opencode-go/deepseek-v4-flash",
  "enabled_providers": ["opencode-go"]
}
```

但 cc-connect 的配置里又写了一行：

```toml
model = "opencode/deepseek-v4-flash-free"
```

结果是：

```text
直接运行 OpenCode：正常
通过微信触发 cc-connect：失败
```

原因是 cc-connect 调用 OpenCode 时使用了它自己配置中的错误模型，覆盖了工作区配置。

修复方式如下：

```toml
[projects.agent.options]
work_dir = "/srv/personal-agent/projects/sandbox"
mode = "default"

# 不要在这里重复写 model
```

原则是：

> cc-connect 只负责“把消息送到哪个 OpenCode project”，不要重复管理模型。

模型应该统一放在项目级：

```text
/srv/personal-agent/projects/sandbox/opencode.jsonc
```

这样未来迁移到公司知识库时也更清晰：

```text
不同知识库 project
  ↓
不同 opencode.jsonc
  ↓
不同模型 / 权限 / Skills
```

---

## 5. 通用 workplace 设计

这次个人 workspace 最终设计为：

```text
sandbox/
├── AGENTS.md
├── opencode.jsonc
├── inbox/
├── work/
├── notes/
├── deliverables/
├── archive/
├── tmp/
└── .opencode/
    └── skills/
```

目录职责如下：

```text
inbox/
  用户上传或待处理文件

work/
  当前任务、实验、小项目

notes/
  调研、分析、过程记录

deliverables/
  最终交付文件

archive/
  已完成或废弃内容

tmp/
  可删除临时文件

.opencode/skills/
  项目级 Skills
```

对公司知识库来说，这个结构也适用，只是命名和内容会更偏业务：

```text
company-kb/
├── AGENTS.md
├── README.md
├── START_HERE.md
├── docs/
├── inbox/
├── raw-notes/
├── processed/
├── deliverables/
└── .opencode/skills/
```

核心思想是：

> 不把 Agent 当成“万能聊天框”，而要给它一个清晰的工作台。或者说, 虽然 Agent 是通用的, 但是承载Agent的工作区, 其实是专用的, 可以弱结构, 但也不能没结构。

---

## 6. AGENTS.md：定义工作区宪法

`AGENTS.md` 是 OpenCode 读取项目规则的主要入口。

对于 IM 接入型 Agent，`AGENTS.md` 应该至少定义：

1. 工作区用途；
2. 工作区边界；
3. 可访问和不可访问目录；
4. 多用户注意事项；
5. 文件放置规范；
6. 任务流程；
7. 交付流程；
8. cc-connect / 微信 / 飞书环境限制。

例如：

```markdown
## Workspace Boundary

The current workspace is the only permitted working area.

Do not access or modify:

- /etc
- /root
- /opt
- unrelated directories under /srv
- WireGuard configuration
- SillyTavern files
- cc-connect configuration
- OpenCode authentication files
- SSH configuration
- firewall configuration
- systemd services

Do not use sudo, switch users, or attempt privilege escalation.
```

对于公司知识库，边界还应该更严格：

```markdown
Do not modify production services.
Do not expose client data.
Do not move raw documents without explicit instruction.
Do not send confidential material to external tools unless approved.
```

这类规则**不能**完全替代系统权限，但能显著减少 Agent 误操作。

---

## 7. Skills：不要堆满，先做核心工作流

在部署过程中, 也评估了是否直接安装大量社区 Skills。

最终比较稳妥的策略是：

```text
社区 Skills:
  Superpowers 可考虑用于软件开发流程

项目本地 Skills:
  task-intake
  workspace-safety-review
  research-brief
  delivery-review
```

不要一开始安装一大包不明来源 Skills。Skills 太多反而会出现：

- 规则重叠；
- 触发混乱；
- 与当前项目不匹配；
- 老版本流程误导；
- 对简单任务过度工程化。

本地 Skills 更适合承担 workplace 强相关流程。

### 7.1 task-intake

用于较大任务开始前：

```text
识别目标
识别输入
确定输出
创建 work/<task-name>/
记录 TASK.md
选择后续 Skills
```

### 7.2 workspace-safety-review

用于高风险变更前：

```text
批量修改
删除文件
迁移目录
依赖变更
shell-heavy 操作
```

它要求 Agent 先检查：

- 是否在工作区内；
- 是否会覆盖别人文件；
- 是否涉及密钥；
- 是否有 rollback；
- 是否需要用户确认。

### 7.3 research-brief

用于正式调研：

```text
当前信息核查
资料来源记录
事实 / 推理分离
形成 notes/<topic>/brief.md
```

公司知识库场景里，这个 Skill 很适合做：

- 技术选型调研；
- 产品方案对比；
- 客户行业资料整理；
- 内部 FAQ 起草。

### 7.4 delivery-review

用于交付前检查：

```text
文件是否存在
格式是否正确
是否有 README
是否包含临时文件
是否泄露 token
是否验证过
```

对公司知识库尤其重要，因为 Agent 生成的文件很容易混入临时路径、debug 信息或未验证内容。

---

## 8. 权限策略：IM 场景下不要依赖 ask


在终端 TUI 中，OpenCode 可以弹出确认：

```text
Agent wants to run command X.
Allow / Deny?
```

但在微信或飞书里，这个交互不稳定，也不自然。

尤其通过 cc-connect 调用 OpenCode 时，很多 `ask` 权限可能变成：

```text
需要确认
  ↓
IM 端没有完整确认回路
  ↓
工具调用失败
  ↓
任务中止
```

因此，IM 接入型 OpenCode 项目更适合采用：

```text
明确安全的操作：allow
明确危险的操作：deny
尽量少用 ask
```

例如：

```jsonc
"permission": {
  "*": "allow",

  "external_directory": "deny",
  "question": "deny",

  "bash": {
    "*": "allow",

    "sudo *": "deny",
    "systemctl *": "deny",
    "service *": "deny",
    "journalctl *": "deny",

    "apt *": "deny",
    "apt-get *": "deny",
    "docker *": "deny",

    "rm -rf *": "deny",
    "git reset --hard*": "deny",
    "git clean *": "deny",
    "git push --force*": "deny",

    "cc-connect daemon *": "deny",
    "cc-connect weixin *": "deny"
  }
}
```


> 微信和飞书不是权限审批终端，而是任务入口。  
> **权限应该预先设计好，不应该在每次工具调用时临时询问**。

当 Agent 需要用户决策时，应该普通回复并停止, 而不是事件回调：

```text
这里需要你选择 A 或 B。我暂时不继续修改。
```

等用户下一条消息确认后再继续。

---

## 9. 文件上传：附件应复制到 inbox，而不是依赖临时路径

微信和飞书都支持上传附件，但通过 cc-connect 进入 OpenCode 时，附件路径往往是临时的。

更合理的规则是：

```text
用户上传附件
  ↓
cc-connect 临时接收
  ↓
Agent 在本次任务中读取
  ↓
如需长期使用，复制到 workspace/inbox/
```

建议在 `AGENTS.md` 中增加：

```markdown
## Incoming Attachments

Attachments received from cc-connect may be temporary.

When an attachment will be needed beyond the current request:

1. Copy it into `inbox/YYYY-MM-DD/`.
2. Preserve the original extension.
3. Sanitize unsafe filename characters.
4. Never overwrite an existing file.
5. Report the saved path, file size, and SHA256.
6. Treat the saved copy as the stable input for subsequent work.
```

实际使用时可以这样对 Agent 说：

```text
请把我刚上传的附件复制到 inbox/2026-06-11/，
保留原始文件名，不要覆盖已有文件。
复制完成后告诉我保存路径、文件大小和 SHA256。
然后再开始分析。
```

这条规则可能对公司知识库更重要。

公司场景中，上传文件往往不是一次性聊天材料，而是知识库的原始输入。它们应该进入稳定目录，而不是散落在平台临时缓存里。

---

## 10. 微信与飞书的体验差异

### 10.1 微信

微信适合：

```text
远程派任务
快速问答
上传小文件
看最终结果
让 Agent 做简单整理
```

但微信不是很适合：

```text
长时间流式输出
复杂权限确认
大量 diff 阅读
频繁选择分支
复杂调试 steering
```


### 10.2 飞书

飞书更适合公司内部 Agent：

```text
群聊 @bot
线程隔离
卡片进度
团队协作
文件收发
消息沉淀
```

飞书 Bot 可以加入群聊，但建议保持：

```toml
group_reply_all = false
```

也就是只响应 `@机器人` 的消息，而不是监听群内所有聊天。

否则会出现：

- 误触发；
- 成本不可控；
- 隐私边界模糊；
- 群聊噪音变大。

更合理的群聊使用方式是：

```text
群里讨论
  ↓
@Agent 分配明确任务
  ↓
Agent 输出结果或文件
  ↓
复杂修改转入私聊或 OpenCode TUI
```

---

## 11. 多用户与多项目

cc-connect 支持一个进程管理多个项目：

```toml
[[projects]]
name = "user-a"

[[projects]]
name = "user-b"

[[projects]]
name = "company-kb"
```

每个项目可以有：

- 独立 agent；
- 独立工作目录；
- 独立平台；
- 独立用户权限。

这让轻量多用户成为可能。

例如：

```text
/srv/personal-agent/projects/
├── john/
├── friend-a/
├── friend-b/
└── shared/
```

或者公司场景：

```text
/srv/company-agents/
├── kb-docs/
├── sales-copilot/
├── support-faq/
└── demo-workspace/
```

但要注意：

> 多项目不等于强隔离。

如果多个项目仍以同一个 Linux 用户运行，那么它们本质上共享该用户的系统权限。  
`work_dir` 是逻辑隔离，不是内核级隔离。

因此这套模式适合：

```text
可信用户
内部团队
低风险任务
有 Git 恢复
有权限 deny
有日志审计
```

不适合：

```text
陌生用户
外部公开服务
强安全隔离要求
高风险生产操作
```

真正不可信用户场景，仍然应该考虑：

- 不同 Linux 用户；
- 不同 cc-connect 实例；
- systemd 用户服务；
- Docker / Podman；
- 容器资源限制；
- 独立卷；
- 只读根文件系统；
- 网络限制。

---

## 12. allow_from 与 admin_from：谁能触发，谁能管理

在 cc-connect 中，要区分两个概念：

```text
allow_from
= 谁的消息可以触发 Agent

admin_from
= 谁可以使用管理命令
```

例如：

```toml
[[projects]]
name = "personal-sandbox"
admin_from = "你的微信ID,你的飞书ID"
```

这只代表这些用户是管理员。

平台层还需要：

```toml
[[projects.platforms]]
type = "weixin"

[projects.platforms.options]
allow_from = "你的微信ID"
```

以及：

```toml
[[projects.platforms]]
type = "feishu"

[projects.platforms.options]
allow_from = "你的飞书ID"
allow_chat = "*"
group_reply_all = false
```

这样才能保证：

```text
只有 allow_from 中的用户能触发 Agent
只有 admin_from 中的用户能使用管理命令
群聊中只有 @bot 才响应
```

最容易犯的错误是：

```toml
admin_from = "你的ID"
```

但平台层没有配置 `allow_from`。

这种情况下，其他人虽然不是管理员，但仍可能触发普通 Agent 任务。

所以对公司 Agent 来说，至少要有两层控制：

```text
平台 allow_from / allow_chat
  ↓
项目 admin_from
  ↓
OpenCode permission
  ↓
AGENTS.md / Skills
```

---

## 13. 适合公司的接入模式

把这次个人部署抽象出来，公司内部 Agent 可以采用类似模式：

```text
Feishu Group / Private Chat
  ↓
cc-connect
  ↓
OpenCode Project
  ↓
Company KnowledgeBase Workspace
  ↓
AGENTS.md + Skills + Permissions
```

但公司版本应进一步加强：

### 13.1 项目隔离

不同用途拆成不同项目：

```text
company-kb
sales-demo
support-faq
data-analysis
admin-sandbox
```

不要把所有任务塞进一个 workspace。

### 13.2 用户权限

不同项目限制不同用户：

```text
知识库维护人员
  可以编辑 docs/

普通业务用户
  只能提问和生成草稿

管理员
  可以执行维护命令
```

### 13.3 审计

至少记录：

- 谁触发；
- 何时触发；
- 来自哪个平台；
- 哪个 project；
- 修改了哪些文件；
- 是否生成 deliverable；
- 是否运行测试或校验。

### 13.4 数据边界

公司知识库常涉及客户文档、业务数据和内部流程，因此要明确：

- 哪些文件可读；
- 哪些文件可改；
- 哪些内容不能发送给外部模型；
- 哪些输出必须人工审核；
- 原始文件如何归档；
- 生成文件如何交付。

### 13.5 交付目录

建议所有最终文件都进入：

```text
deliverables/<task-name>/
```

并包含：

```text
README.md
source/
output/
validation-notes.md
```

这比让 Agent 在聊天里吐一大段内容更可控。

---

## 14. 这套方式的边界

这套架构很好用，但不能神化。

它适合：

```text
远程派活
知识库维护
文档整理
内部问答
小型代码修改
批量格式处理
轻量数据分析
生成报告
```

不适合直接承担：

```text
生产系统运维
高风险服务器操作
不可信用户沙箱
强权限审批流程
复杂长时间 debug
大量人工 steering
```

微信和飞书是优秀的入口，但不是完整终端。  
复杂任务仍然应该回到：

```text
SSH + OpenCode TUI
```

或者：

```text
Web UI / IDE / 专门的 Agent 控制台
```

更准确的定位是：

> IM 平台负责派发任务和接收结果；  
> OpenCode 负责执行；  
> workspace 负责沉淀；  
> AGENTS.md、Skills 和 permission 负责约束；  
> Git 和日志负责恢复与审计。

---

## 15. 小结

这次个人部署给我的核心结论：

1. **cc-connect 可以把 OpenCode 从终端扩展到微信、飞书等协作平台。**
2. **模型配置应该放在 OpenCode 项目里，不要在 cc-connect 里重复指定。**
3. **IM 场景不适合大量使用 `ask` 权限；应该采用 allow / deny 的确定性策略。**
4. **附件要复制到 workspace 的 `inbox/`，不要依赖平台临时路径。**
5. **微信更像远程遥控器，飞书更适合公司协作入口。**
6. **多项目可以支持轻量多用户，但不是强安全隔离。**
7. **公司内部 Agent 可以复用这套架构，但必须加强权限、审计和数据边界。**

所以，这篇虽然来自个人服务器部署，但它也讨论的是一个更通用的问题：

> 如何把 OpenCode 从“开发者本地工具”变成“团队协作平台里的 Agent Runtime”。
