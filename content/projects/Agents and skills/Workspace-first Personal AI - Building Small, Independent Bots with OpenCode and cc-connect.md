---
title: Workspace-first Personal AI - Building Small, Independent Bots with OpenCode and cc-connect
tags:
  - ai-agent
  - opencode
  - cc-connect
  - workspace
  - self-hosting
draft: false
created: 2026-10-08 00:00
modified: 2026-10-08 23:23
---

一台 Linux 服务器可以同时运行多个 AI Bot：一个处理文书，一个阅读论文，一个负责日常对话或图片生成。通过 Telegram、飞书或微信发送任务，由 OpenCode 访问各自的工作目录并执行操作。

[[Conversation First or Workspace First]] 讨论了以 Workspace 为工作中心的取舍。这篇继续整理多个个人 Bot 的部署方式。

这套方案采用以下组件：

- **OpenCode**：Agent Runtime，负责模型调用、工具执行和 Session。
- **cc-connect**：连接 IM 平台与 OpenCode，将不同 Bot 的消息路由到对应 Project。
- **Filesystem + Git**：保存工作规则、资料、脚本、进度和交付文件。
- **AGENTS.md / Skills**：为不同 Workspace 定义职责与可复用能力。

不引入中心调度 Agent、长期对话 Memory、向量数据库或额外的工作流引擎。

## 1. Architecture

### Workspace-first

这套架构将 Workspace 作为长期工作单元，Chat Session 承载当前交互的上下文。

```text
Telegram / Feishu / Weixin
             |
         cc-connect
             |
       OpenCode Runtime
             |
    +--------+--------+
    |        |        |
  Writer   Papers    Pufi
    |        |        |
   repo     repo     repo
```

每个 Workspace 都是普通的文件目录，可以独立初始化为 Git Repository。

```text
~/ai-workspaces/
├── writer/
│   ├── AGENTS.md
│   ├── STATUS.md
│   ├── inbox/
│   ├── drafts/
│   └── outputs/
│
├── papers/
│   ├── AGENTS.md
│   ├── sources/
│   ├── notes/
│   └── STATUS.md
│
└── pufi/
    ├── AGENTS.md
    ├── .opencode/
    │   └── skills/
    └── outputs/
```

这里需要区分三个概念：

| 概念 | 职责 |
|---|---|
| Bot | 用户通过 IM 访问某个工作环境的入口 |
| Session | 一次或一组连续交互的上下文 |
| Workspace | 持久化的项目资料、规则、工具和工作成果 |

Bot 与 Workspace 不必严格一对一。同一个 Workspace 可以提供不同的 IM 入口；一个 cc-connect 进程也可以管理多个 Project。

### 不依赖长期对话 Memory

OpenCode 本身仍然管理和保存 Session。这里不使用 Session 作为长期工作状态的权威来源，也不建立独立的 Memory 系统。

如果 Paper Bot 已经处理了十篇论文，它应该保存结构化笔记、来源和阅读进度，而不只是让模型在上下文中记住论文内容。

重新开始一个 Session 时：

1. OpenCode 加载项目级 `AGENTS.md`。
2. Agent 根据任务读取 `STATUS.md`、索引和相关文件。
3. 在已有文件基础上继续工作。
4. 将新增结果写回 Workspace。

需要复用的信息写入文件，供后续 Session 按任务读取。

### 为什么使用 Git

Git 管理工作资产。Agent Runtime 的安装、配置和状态需要单独管理。

适合放入 Git 的内容包括 Markdown、规则文件、Skills、脚本和小型结构化数据。API Key、Session 数据库、大型 PDF 和私人附件则需要分别处理。

这也减少了运行时绑定：即使将来更换模型或 Agent 工具，已有项目文件仍可以继续使用。

## 2. Environment

本文采用：

| Component | Choice |
|---|---|
| OS | Ubuntu 24.04 LTS |
| Runtime | OpenCode v2 |
| IM Bridge | cc-connect |
| Storage | Local filesystem + Git |
| Model | 用户自行配置的 LLM Provider |
| Primary IM | Telegram |
| Optional IM | Feishu / Weixin |

服务器不需要 GPU，因为模型推理可以通过远程 API 完成。资源消耗主要来自 OpenCode、工具调用和实际运行的脚本。

本文采用普通 Linux 用户的 Home 目录作为工作区，不需要 Docker，也不需要公网开放 HTTP 服务。

以下按 OpenCode v2 文档和 cc-connect 当前配置整理，核对日期为 2026-10-08；本文未提供一组经过端到端验证的具体版本号。部署时记录两者的 `--version`，先完成 Writer 的模型调用、文件写入和 IM 回复验证，再扩展多个 Project。v1 与 v2 的权限格式不同，不能直接混用。

### 安装 OpenCode v2

使用 OpenCode v2 的安装入口：

```bash
curl -fsSL https://opencode.ai/v2/install | bash

opencode --version
```

然后配置 LLM Provider：

```bash
opencode
```

在 TUI 中使用 `/connect`，按提示设置模型提供商和认证信息。

确认模型可以正常调用：

```bash
opencode models

opencode run --standalone "只回复 OPENCODE_OK"
```

使用 `--standalone` 进行独立测试，避免和已经运行的共享 OpenCode 服务混淆。

OpenCode v2 也提供共享后台服务：

```bash
opencode service start
opencode service status
```

同一台机器通常不需要为每个 Bot 启动独立的 OpenCode Server。Session 和 Project 可以在同一 Runtime 下保持逻辑独立。

共享服务默认按用户账户运行，`--standalone` 使用私有 server，参见 [OpenCode v2 CLI](https://opencode.ai/v2/docs/cli/)。cc-connect 的 OpenCode adapter 通过 `opencode run --format json` 调用 CLI；部署时还需检查所用 CLI 是否接受 adapter 传入的参数，以及工具权限请求如何处理。

### 安装 cc-connect

安装好 Node.js 和 npm 后：

```bash
npm install -g cc-connect

cc-connect --version
```

也可以使用 [cc-connect Releases](https://github.com/chenhg5/cc-connect/releases) 中的预编译二进制。

本文采用 TOML 配置文件，不使用 Web 管理界面。

## 3. First Workspace：Writer Bot

先建立一个简单的文书工作区。

```bash
mkdir -p ~/ai-workspaces/writer/{inbox,drafts,outputs}

cd ~/ai-workspaces/writer
git init
```

Writer 只负责整理文档、修改草稿和生成 Markdown 文件，不承担系统管理或软件部署任务。

### AGENTS.md

创建 `~/ai-workspaces/writer/AGENTS.md`：

```markdown
# Writer Workspace

You are a personal document assistant.

## Responsibilities

- Organize notes and rough drafts.
- Rewrite and edit Markdown documents.
- Prepare short reports and messages.
- Preserve source material and factual accuracy.

## Workspace

- inbox/: original input files
- drafts/: editable working documents
- outputs/: final deliverables
- STATUS.md: ongoing work and next actions

## Rules

- Work within this workspace by default.
- Do not modify files in inbox/.
- Do not invent missing facts or sources.
- Read existing drafts before creating duplicates.
- Keep output concise and structured.
- Save useful work to files, not only chat replies.
- Update STATUS.md when ongoing work changes.

Do not execute system administration commands.
```

`AGENTS.md` 是 Workspace 的项目级指令文件。OpenCode v2 会读取它，不必每次通过聊天消息重新提供相同的规则。

但它不是 Linux 权限控制，不能仅依赖这份文件阻止危险操作。

### STATUS.md

初始化一个非常简单的状态文件：

将下面内容保存为 `~/ai-workspaces/writer/STATUS.md`：

```markdown
# Current Work

## Active Tasks

None.

## Important Files

None.

## Next Actions

None.
```

`STATUS.md` 不是 Memory Database，也不需要记录完整的聊天过程。

它只保存项目当前的工作状态。对于没有持续任务的 Workspace，这个文件甚至可以省略。

### 本地验证

```bash
cd ~/ai-workspaces/writer

opencode run --standalone \
  "Read AGENTS.md and create drafts/hello.md with a short Markdown introduction to this workspace."
```

完成后检查：

```bash
ls -l drafts/
git status --short
```

确认 OpenCode 能读取规则并生成文件后，再连接 IM。这样可以把模型和工具问题与消息接入问题分开排查。

## 4. Connect Telegram

在 Telegram 中打开官方 [@BotFather](https://t.me/BotFather)。

使用 `/newbot` 创建一个 Bot，保存 Token，并获取自己的 Telegram User ID。

创建 cc-connect 配置：

```bash
mkdir -p ~/.cc-connect
chmod 700 ~/.cc-connect

nano ~/.cc-connect/config.toml
```

最小配置：

```toml
language = "zh"

[log]
level = "info"

[[projects]]
name = "writer"

[projects.agent]
type = "opencode"

[projects.agent.options]
work_dir = "/home/YOUR_USER/ai-workspaces/writer"
mode = "default"

[[projects.platforms]]
type = "telegram"

[projects.platforms.options]
token = "YOUR_WRITER_BOT_TOKEN"
allow_from = "YOUR_TELEGRAM_USER_ID"
```

将 `YOUR_USER`、Bot Token 和 Telegram User ID 替换为实际值。`work_dir` 使用绝对路径。

`allow_from` 限制可以向 Bot 下达任务的用户。项目级 `admin_from` 控制 cc-connect 特权管理命令；这里不配置它，默认不开放相关管理权限。

这只限制 cc-connect 自己的管理命令，Agent 工具的执行权限仍由 OpenCode 配置决定。`mode = "default"` 也不构成操作系统隔离；跨目录操作可能需要授权，验收时应同时检查日志中的权限请求和实际文件结果。

不建议让公开可访问的 Bot 接受任何用户的工具执行请求。

配置文件包含 Token，应限制权限：

```bash
chmod 600 ~/.cc-connect/config.toml
```

现在启动：

```bash
cc-connect -config ~/.cc-connect/config.toml
```

在 Telegram 中向 Writer Bot 发送：

> 在 drafts/ 目录下新建 test.md，写一份简单的 Markdown Todo List，然后告诉我保存路径。

随后通过 SSH 查看：

```bash
cat ~/ai-workspaces/writer/drafts/test.md
git -C ~/ai-workspaces/writer status --short
```

这里通过实际生成的文件确认消息触发了正确 Workspace 内的操作。

## 5. From One Bot to Multiple Workspaces

cc-connect 的 `[[projects]]` 可以重复配置。一个进程即可同时管理多个 Project，每个 Project 有自己的 `work_dir`、Agent 设置和 IM 入口。

### Papers Workspace

创建第二个工作区：

```bash
mkdir -p ~/ai-workspaces/papers/{sources,notes}
cd ~/ai-workspaces/papers
git init
```

`AGENTS.md` 可以更偏向研究资料管理：

```markdown
# Papers Workspace

You assist with reading and organizing
academic papers.

## Workflow

1. Identify the source document.
2. Extract the title, authors, year, and DOI
   when available.
3. Summarize the research question,
   methodology, results, and limitations.
4. Record important claims with source
   references and page numbers when available.
5. Save notes under notes/.
6. Update STATUS.md when reading progress changes.

## Rules

- Separate source claims from interpretation.
- Never fabricate citations.
- Mark missing or uncertain information.
- Preserve source metadata.
- Do not silently overwrite previous notes.
- Do not modify unrelated workspaces.
```

论文处理不一定需要复杂的 RAG 系统。

对于个人使用，可以先从直接读取 PDF、提取文本、整理 Markdown 笔记开始。若使用 `pdftotext` 等工具，需要另外安装对应程序并配置执行权限；扫描版 PDF 还可能需要 OCR。

处理大量文献时，再按检索需求增加全文索引或向量检索。

### Pufi Workspace

第三个工作区可以采用现有的 [pufi-public](https://github.com/John-Ferrel/pufi-public)。

Pufi 是一个带 persona 的个人助手，并包含两个图片生成 Skill：

- `pufi-image`：自然语言图像生成。
- `pufi-anime`：偏动漫场景的图像生成。

公开仓库已经包含安装和基本检查脚本：

```bash
mkdir -p ~/ai-workspaces/pufi/outputs

git clone https://github.com/John-Ferrel/pufi-public.git
cd pufi-public

./install.sh --target custom --opencode-dir "$HOME/ai-workspaces/pufi/.opencode" --yes
./install.sh doctor
```

这里将公开仓库作为安装源，安装目标明确指向 `~/ai-workspaces/pufi/.opencode`，与下文 `work_dir` 一致。参数依据见 [Pufi README](https://github.com/John-Ferrel/pufi-public#-常用命令)。图片生成还需要根据 README 配置 API Key，并检查当前 OpenCode 版本下的实际兼容情况。

安装 persona 和 Skills 后，还要确认 Pufi Workspace 实际选择了对应 Agent。可通过 CLI 的 Agent 列表核对 ID，再设置项目默认 Agent，或在 cc-connect 的 `[projects.agent.options]` 中指定 `agent`；下面的通用多项目配置未指定 persona ID。工作区树中的 `AGENTS.md` 也需要按自己的职责创建，不能假定安装器会生成这份文件。

Pufi 的作用与 Writer、Papers 不同。它可以有更明显的 persona、回复风格和专用 Skill，但不应该默认继承论文阅读或文书工作区的全部规则。

### cc-connect 多项目配置

分别为 Writer、Papers、Pufi 创建独立的 Telegram Bot Token。

将 `~/.cc-connect/config.toml` 扩展为：

```toml
language = "zh"

[log]
level = "info"

# Writer

[[projects]]
name = "writer"

[projects.agent]
type = "opencode"

[projects.agent.options]
work_dir = "/home/YOUR_USER/ai-workspaces/writer"
mode = "default"

[[projects.platforms]]
type = "telegram"

[projects.platforms.options]
token = "WRITER_BOT_TOKEN"
allow_from = "YOUR_TELEGRAM_USER_ID"


# Papers

[[projects]]
name = "papers"

[projects.agent]
type = "opencode"

[projects.agent.options]
work_dir = "/home/YOUR_USER/ai-workspaces/papers"
mode = "default"

[[projects.platforms]]
type = "telegram"

[projects.platforms.options]
token = "PAPERS_BOT_TOKEN"
allow_from = "YOUR_TELEGRAM_USER_ID"


# Pufi

[[projects]]
name = "pufi"

[projects.agent]
type = "opencode"

[projects.agent.options]
work_dir = "/home/YOUR_USER/ai-workspaces/pufi"
mode = "default"

[[projects.platforms]]
type = "telegram"

[projects.platforms.options]
token = "PUFI_BOT_TOKEN"
allow_from = "YOUR_TELEGRAM_USER_ID"
```

这里采用独立的 Bot Token，避免同一 Telegram Bot 同时被多个 Project 争用消息接收。

各 Workspace 的默认模型可以在对应的 OpenCode 项目配置中单独选择，例如：

```json
{
  "$schema": "https://opencode.ai/config.json",
  "model": "provider/model-name"
}
```

实际模型 ID 可以通过 `opencode models` 查询。不要直接将示例中的 `provider/model-name` 用于运行。

cc-connect 不需要再覆盖 OpenCode 的模型选择。

## 6. Working with Files Instead of Memory

下面使用 Papers 与 Writer 展示两个独立工作区如何延续任务。

### Step 1：Papers 处理资料

向 Papers Bot 发送一篇论文，或者先把 PDF 放入：

```text
~/ai-workspaces/papers/sources/
```

任务示例：

> 阅读 sources/example.pdf，整理研究问题、方法、主要发现和局限性。尽可能附上页码，将结果保存为 notes/example.md，并更新 STATUS.md。

预期的文件结构：

```text
papers/
├── sources/
│   └── example.pdf
├── notes/
│   └── example.md
└── STATUS.md
```

这一步的结果应保存在文件中，而不仅是 Telegram 的回复。

### Step 2：启动一个新 Session

在 Papers Bot 中执行：

```text
/new
```

然后发送：

> 检查当前论文阅读进度，告诉我已经完成了什么，以及还有哪些需要补充。

Agent 应读取工作区文件，根据其中的记录回答。

可以进一步要求它列出实际读取过的文件，核对进度结论的来源。

### Step 3：Writer 读取 Papers 的成果

Writer 和 Papers 默认是不同的 Project，不共享 Chat Session。

但是它们运行在同一台 Linux 服务器上，因此可以在明确授权后直接读取彼此的文件。

对于 OpenCode v2，可以在 Writer 的 `opencode.jsonc` 中配置跨目录权限：

```json
{
  "$schema": "https://opencode.ai/config.json",
  "permissions": [
    {
      "action": "external_directory",
      "resource": "*",
      "effect": "deny"
    },
    {
      "action": "external_directory",
      "resource": "$HOME/ai-workspaces/papers/notes/*",
      "effect": "allow"
    },
    {
      "action": "read",
      "resource": "$HOME/ai-workspaces/papers/notes/*",
      "effect": "allow"
    },
    {
      "action": "edit",
      "resource": "$HOME/ai-workspaces/papers/notes/*",
      "effect": "deny"
    }
  ]
}
```

上述权限允许 Writer 的文件工具读取 Papers 的 `notes/`，但不允许通过编辑工具修改其中的文件。

v2 会展开路径开头的 `$HOME`，规则按顺序匹配，最后一个匹配项生效。已有的全局或 Agent 规则也会影响最终结果，参见 [OpenCode v2 Permissions](https://opencode.ai/v2/docs/permissions/)。

这是 OpenCode 层面的权限，不是操作系统级沙箱。Shell 仍然拥有运行该进程的 Linux 用户权限，实际部署时应另行限制不需要的 Shell 操作。

在 Writer Bot 中发送：

> 读取 /home/YOUR_USER/ai-workspaces/papers/notes/example.md，根据这篇论文的笔记写一篇简短的技术介绍，保存到 drafts/example-review.md。不要修改 Papers Workspace。

将 `YOUR_USER` 替换为实际用户名。Writer 的工作目录是 `writer/`，示例使用绝对路径定位相邻的 Papers Workspace。

任务完成后，Writer 将文章保存在自己的 Workspace，Papers 的笔记保持原样。

无需让 Papers Bot 将历史会话传递给 Writer，也无需建立 Agent-to-Agent 通信协议。

### Step 4：继续文稿修改

在 Writer Bot 中执行 `/new`，然后发送：

> 继续修改 drafts/example-review.md，检查已有内容，压缩重复描述，保留来源信息。更新 STATUS.md。

如果新 Session 能正确读取文件并继续任务，就说明该任务的持续性没有建立在旧会话历史之上。

当然，这不保证 Agent 自动理解所有隐含意图。重要的任务约束仍然需要写入规则、草稿或状态文件。

## 7. Adding Feishu and Weixin

Telegram 只是一个接入入口。同一个 cc-connect Project 可以增加其他平台。

### Feishu

可以使用 cc-connect 的设置命令：

```bash
cc-connect feishu setup --project writer
```

也可以手动创建飞书应用，配置 Bot 能力、消息权限与 WebSocket 事件订阅，再将 `app_id`、`app_secret` 写入对应 Project。

飞书接入采用 WebSocket 长连接，不需要为 Bot 暴露公网 Webhook。

配置参考：[cc-connect Feishu Guide](https://github.com/chenhg5/cc-connect/blob/main/docs/feishu.md)。

对于群聊，我更倾向于仅在明确 @Bot 时触发。Thread 与群聊上下文也应独立处理，相关行为见 [[Feishu Bot + History - cc-connect Group Chat Context]]。

### Weixin

个人微信使用 ilink Bot 接入方式：

```bash
cc-connect weixin setup --project pufi
```

按提示扫码绑定，检查生成的 Token、账号与 `allow_from`。

配置参考：[cc-connect Weixin Guide](https://github.com/chenhg5/cc-connect/blob/main/docs/weixin.md)。

需要注意，微信个人号、企业微信和飞书不是同一套接口，文件消息、卡片渲染及群聊支持也存在差异。

多个平台指向相同的 Workspace，意味着它们可以读取相同的项目文件，**不意味着它们自动共享 Chat Session**。

这正符合本方案的状态管理方式：需要共享的成果保存在 Workspace，聊天上下文由各个平台入口和 Session 自己管理。

## 8. Running as a Service

交互测试通过后，再把 cc-connect 作为长期服务运行。

使用官方 daemon 管理：

先停止前面用于交互验证的 cc-connect 进程，避免同一个 Telegram Bot 被两个进程同时轮询。下面的 `--work-dir` 指包含 `config.toml` 的目录，各 Agent 的工作目录仍由配置中的 `work_dir` 决定。

```bash
cc-connect daemon install --work-dir "$HOME/.cc-connect"

cc-connect daemon start
cc-connect daemon status
```

对于用户级 systemd 服务，需要确保退出 SSH 后仍然运行：

```bash
sudo loginctl enable-linger "$USER"
```

查看日志：

```bash
cc-connect daemon logs -f
```

升级或故障排查时，先分别确认：

```bash
opencode --version
opencode service status

cc-connect --version
cc-connect daemon status
```

如果直接运行 OpenCode 正常，而 IM Bot 不响应，优先检查 cc-connect 日志、平台 Token、`allow_from`、Project 的 `work_dir`，以及 daemon 的 PATH 环境。

特别是通过 Shell 安装的 OpenCode，交互式 SSH 能找到二进制，并不代表 systemd 服务也能找到。必要时检查服务运行用户和实际 PATH。

systemd PATH 的检查方法也见 [[Ubuntu 24.04 上运行 cc-connect + Codex：PATH、Sandbox 与 Git#1. daemon 找不到 Codex]]；其中 Codex 的安装目录需要按实际 OpenCode 路径替换。

OpenCode v2 的共享服务还需要注意端口冲突。不要让多个独立进程争用同一个后台服务端口，也不要在没有确认兼容方式的情况下同时启动多个 Runtime。

## 9. Storage, Backup and Migration

每个 Workspace 独立管理 Git：

```bash
cd ~/ai-workspaces/writer

git add AGENTS.md STATUS.md drafts/
git commit -m "docs: initialize writer workspace"
```

对于个人资料，建议按工作性质区分 Git 管理范围。

例如：

```gitignore
.env
.env.*
inbox/
tmp/
*.pdf
```

这只是示例。需要版本化的 PDF 可以单独纳入管理；不适合进入 Git 的大文件应使用普通文件备份。

Git 不等于完整备份。至少还需要保存：

- 不在 Git 中的源文件与附件。
- OpenCode、cc-connect 的必要配置。
- 可以安全恢复的 API 凭证或重新签发方式。
- Workspace 的目录对应关系。

迁移到其他 Linux 服务器时，不需要迁移完整的聊天历史才能继续使用这些 Workspace。

基本恢复流程是：

1. 安装 OpenCode 与 cc-connect。
2. 恢复模型认证和 IM 平台凭证。
3. Clone 或复制各 Workspace。
4. 恢复未纳入 Git 的资料。
5. 更新 `work_dir` 和跨目录权限。
6. 重新验证模型调用、文件操作与 IM 接入。

Agent Runtime、聊天平台和模型服务都可以在此过程中更换，但迁移并不保证旧 Session、插件或工具配置完全兼容。Git 保存项目内容，完整运行时状态需要另行备份。

## 10. Limitations

这套结构面向个人可信环境，并不解决所有 Agent 运维问题。

**隔离不是强安全边界。** 多个 OpenCode Project 使用不同的工作目录，但如果都由同一个 Linux 用户运行，仍可能拥有相同的文件和进程权限。`AGENTS.md` 与 OpenCode Permissions 不能完全替代 Unix 权限、容器或其他沙箱机制。

**文件状态需要主动维护。** Agent 如果没有将重要结果保存为文件，新的 Session 就可能失去必要上下文。`STATUS.md` 也必须保持准确，不能将过时的状态记录当作事实。

**跨 Workspace 访问需要控制。** 对少数个人项目，显式读取其他目录通常比引入中心调度服务简单。但如果工作区之间存在不同的信任级别，就应该通过 Linux 用户、只读导出目录或更强的隔离方式处理。

**IM 不适合所有交互。** 长时间执行、复杂工具授权、大文件传输和持续调试，往往仍然更适合 SSH 或 OpenCode TUI。IM 主要用来随时发起简单任务。

**仍然依赖外部平台。** OpenCode、cc-connect、Git 和工作区文件可以自行管理；LLM Provider、Telegram、飞书、微信的服务与协议仍可能受第三方控制。工作资产可以迁移，外部服务仍需要单独评估。

---

## References

- [OpenCode v2 Documentation](https://opencode.ai/v2/docs/)
- [OpenCode v2 Permissions](https://opencode.ai/v2/docs/permissions/)
- [cc-connect](https://github.com/chenhg5/cc-connect)
- [cc-connect Configuration Example](https://github.com/chenhg5/cc-connect/blob/main/config.example.toml)
- [Pufi Public Repository](https://github.com/John-Ferrel/pufi-public)
- [[Building KnowledgeBase with OpenCode 3 - CC Connect]]
- [[Agent and Skills Development]]
