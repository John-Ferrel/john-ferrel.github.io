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
modified: 2026-07-04 00:00
---

前两篇 [[Building KnowledgeBase with Opencode]] 和 [[Building KnowledgeBase with OpenCode 2 - Docker and SFTPGo]] 主要讨论如何把 OpenCode project / workspace 用作知识库工程化工作台。

这一篇继续往外扩一步：用 cc-connect 把 OpenCode 接到微信 / 飞书，让 Agent 可以从 IM 入口接收任务。

记录一条最小可用路径：

```text
微信 / 飞书
  ↓
cc-connect
  ↓
OpenCode project
  ↓
workspace
  ↓
AGENTS.md / Skills / opencode.jsonc
```

参考资料：

* [cc-connect README](https://github.com/chenhg5/cc-connect)
* [cc-connect INSTALL.md](https://github.com/chenhg5/cc-connect/blob/main/INSTALL.md)
* [飞书接入指南](https://github.com/chenhg5/cc-connect/blob/main/docs/feishu.md)
* [微信个人号接入指南](https://github.com/chenhg5/cc-connect/blob/main/docs/weixin.md)
* [config.example.toml](https://github.com/chenhg5/cc-connect/blob/main/config.example.toml)

## 1. 目标

目标：让微信 / 飞书成为 OpenCode project 的任务入口。

验收标准：

```text
1. 微信或飞书可以触发 OpenCode
2. cc-connect 根据消息来源路由到指定 project
3. OpenCode 在固定 work_dir 内执行任务
4. 模型、权限、Skills 由 OpenCode project 管理
5. 附件或临时文件进入 workspace/inbox/
6. 最终结果进入 workspace/deliverables/
```

链路分工很窄：cc-connect 做消息接入、用户识别和 project 路由；OpenCode 执行任务；workspace 沉淀输入、过程文件和交付物；`AGENTS.md` / Skills / `opencode.jsonc` 约束 Agent 行为。

核心判断：cc-connect 只做路由层。业务逻辑、模型策略和执行边界留在 OpenCode project。

## 2. 部署形态

我的个人服务器资源：

```text
CPU: 4 核
内存: 4GB
系统盘: 40GB SSD
带宽: 3Mbps
系统: Ubuntu Server 24.04 LTS
```

这个配置足够运行 cc-connect、OpenCode、微信 / 飞书连接和远程 LLM API 调用。本地大模型不放在这台机器上。

第一版采用轻量方案：

```text
Linux user: agent
  ↓
cc-connect 进程
  ↓
一个或多个 projects
  ↓
每个 project 指向一个 work_dir
```

这类方案适合可信用户、小团队、低风险的文档 / 代码 / 研究类工作。陌生用户、生产运维、高敏数据和不可信代码执行，要继续上 Linux user 或容器隔离。

隔离等级大致是：

```text
session
  只隔离聊天上下文，文件全都共用

project / work_dir
  每个项目一个目录，AGENTS.md / opencode.jsonc / Skills 分开

Linux user
  不同 Unix 用户，靠系统权限隔离

Docker / Podman
  容器级隔离，适合更不可信的场景
```

本文采用：

```text
project / work_dir 级别的轻量隔离
```

## 3. 推荐目录

统一使用一个基础目录：

```text
/srv/personal-agent/
├── cc-connect/
│   └── config.toml
├── projects/
│   └── sandbox/
│       ├── AGENTS.md
│       ├── opencode.jsonc
│       ├── inbox/
│       ├── work/
│       ├── notes/
│       ├── deliverables/
│       ├── archive/
│       ├── tmp/
│       └── .opencode/
│           └── skills/
└── logs/
```

创建目录：

```bash
sudo mkdir -p /srv/personal-agent/cc-connect
sudo mkdir -p /srv/personal-agent/projects/sandbox
sudo chown -R agent:agent /srv/personal-agent
```

切换到 `agent` 用户：

```bash
su - agent
```

进入 workspace：

```bash
cd /srv/personal-agent/projects/sandbox
mkdir -p inbox work notes deliverables archive tmp .opencode/skills
git init
```

## 4. 安装与基础验证

安装方式可以用 npm、Homebrew、release binary 或源码构建。这里用 npm：

```bash
npm install -g cc-connect
```

确认命令可用：

```bash
cc-connect --version
opencode --version
```

确认 OpenCode 已经能正常调用模型：

```bash
opencode auth list
opencode models
```

在 workspace 内直接测试 OpenCode：

```bash
cd /srv/personal-agent/projects/sandbox

opencode run --format json \
  "Do not use tools. Reply exactly: OPENCODE_OK"
```

如果这一步失败，不要继续配置 cc-connect。
先解决 OpenCode 的 provider、API key 或 model 问题。

## 5. Workspace 约束：AGENTS.md

IM 入口会降低操作门槛，也会放大误操作风险。workspace 需要先写清楚边界。

创建：

```bash
cd /srv/personal-agent/projects/sandbox
nano AGENTS.md
```

示例：

```markdown
# Personal Agent Workspace

This workspace is the only permitted working area.

## Allowed working directories

- inbox/
- work/
- notes/
- deliverables/
- archive/
- tmp/

## Do not access or modify

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

Do not use sudo.
Do not switch users.
Do not attempt privilege escalation.

## Outputs

When a task creates final output, place it under:

deliverables/<task-name>/

## Attachments

Attachments received from cc-connect may be temporary.

When an attachment or temporary input should be preserved:

1. Copy it into inbox/YYYY-MM-DD/.
2. Preserve the original extension.
3. Do not overwrite existing files.
4. Report the saved path, file size, and SHA256.
5. Use the saved copy as the stable source file.
```

`AGENTS.md` 不能替代系统权限，但能让 OpenCode 在执行前读到项目边界。通过微信 / 飞书触发时，用户通常不会像在 TUI 里逐步确认每个动作。

## 6. OpenCode 配置：opencode.jsonc

模型和权限放在 project 的 `opencode.jsonc`。

创建：

```bash
cd /srv/personal-agent/projects/sandbox
nano opencode.jsonc
```

示例：

```jsonc
{
  "$schema": "https://opencode.ai/config.json",

  "model": "opencode-go/deepseek-v4-flash",
  "small_model": "opencode-go/deepseek-v4-flash",
  "enabled_providers": ["opencode-go"],

  "permission": {
    "*": "allow",

    "read": {
      "*": "allow",

      "*.env": "deny",
      "*.env.*": "deny",
      "*.pem": "deny",
      "*.key": "deny",
      "**/id_rsa": "deny",
      "**/id_ed25519": "deny",
      "**/*token*": "deny",
      "**/*secret*": "deny",
      "**/*credential*": "deny"
    },

    "edit": "allow",
    "glob": "allow",
    "grep": "allow",
    "skill": "allow",
    "lsp": "allow",
    "task": "allow",

    "websearch": "allow",
    "webfetch": "allow",

    "external_directory": "deny",
    "question": "deny",
    "doom_loop": "deny",

    "bash": {
      "*": "allow",

      "sudo *": "deny",
      "su *": "deny",

      "systemctl *": "deny",
      "service *": "deny",
      "journalctl *": "deny",
      "loginctl *": "deny",

      "ufw *": "deny",
      "iptables *": "deny",
      "nft *": "deny",

      "reboot*": "deny",
      "shutdown*": "deny",
      "poweroff*": "deny",

      "mount *": "deny",
      "umount *": "deny",
      "chown *": "deny",
      "chmod -R *": "deny",

      "apt *": "deny",
      "apt-get *": "deny",
      "snap *": "deny",

      "docker *": "deny",
      "podman *": "deny",

      "rm -rf *": "deny",
      "rm -fr *": "deny",

      "git reset --hard*": "deny",
      "git clean *": "deny",
      "git push --force*": "deny",
      "git push -f *": "deny",

      "cc-connect daemon *": "deny",
      "cc-connect weixin *": "deny",
      "cc-connect feishu *": "deny"
    }
  }
}
```

几个取舍：

1. IM 场景里尽量少用 `ask`，否则工具调用容易中止。
2. `question` 设置为 `deny`。需要用户判断时，让 Agent 直接回复问题，然后停止。
3. `external_directory` 设置为 `deny`，但它只是 OpenCode 层权限，不是 Linux 系统级隔离。

保存后测试：

```bash
cd /srv/personal-agent/projects/sandbox

opencode run --format json \
  "Do not use tools. Reply exactly: CONFIG_OK"
```

## 7. cc-connect 配置：config.toml

本文统一使用固定配置路径：

```text
/srv/personal-agent/cc-connect/config.toml
```

创建：

```bash
nano /srv/personal-agent/cc-connect/config.toml
```

最小结构：

```toml
language = "zh"

[log]
level = "info"

[[projects]]
name = "personal-sandbox"
admin_from = "你的微信ID,你的飞书ID"

[projects.agent]
type = "opencode"

[projects.agent.options]
work_dir = "/srv/personal-agent/projects/sandbox"
mode = "default"
```

这里不写 model。

我踩过一个坑：cc-connect 配置里写了：

```toml
model = "opencode/deepseek-v4-flash-free"
```

结果是：

```text
直接运行 OpenCode：正常
通过微信触发 cc-connect：失败
```

原因是 cc-connect 传入的 model 覆盖了 project 的 `opencode.jsonc`。修复原则：

> cc-connect 不写 model。
> 模型由 OpenCode project 自己管理。

## 8. allow_from 和 admin_from

这两个字段最容易混淆：

```text
allow_from
  谁的普通消息可以触发 Agent

admin_from
  谁可以使用 cc-connect 管理命令
```

管理命令包括：

```text
/dir
/shell
/restart
```

只写 `admin_from` 不够。平台层没有 `allow_from` 时，其他人可能仍然能触发普通 Agent 任务。

可以这样理解：

```text
project 层：
  admin_from 控制管理员

platform 层：
  allow_from 控制谁能触发这个平台入口
  allow_chat 控制哪些飞书群 / 聊天可以触发
```

## 9. 微信接入

先在 `config.toml` 里保留 project 基础配置，然后执行：

```bash
cc-connect weixin setup \
  --config /srv/personal-agent/cc-connect/config.toml \
  --project personal-sandbox
```

这个命令会在终端打印二维码或 URL。手机微信确认后，它会把微信平台配置写回 `config.toml`。

生成后检查：

```bash
grep -nA30 'name = "personal-sandbox"' /srv/personal-agent/cc-connect/config.toml
```

微信平台块应该类似：

```toml
[[projects.platforms]]
type = "weixin"

[projects.platforms.options]
token = "ilink_bot_bearer_token"
base_url = "https://ilinkai.weixin.qq.com"
account_id = "personal-weixin"
allow_from = "你的微信用户ID"
```

如果 `allow_from` 为空，或者是 `"*"`，上线前要手动收紧。

首次使用时，还需要从微信端先发一条消息，让 cc-connect 缓存上下文：

```text
你好
```

然后测试：

```text
/new
```

再发送：

```text
不要调用工具，只回复：WEIXIN_OK
```

接入新的微信用户时，通常不需要重新 setup 整个服务：

```text
1. 让新用户能找到这个微信 Bot
2. 临时放开 allow_from 或让他发送 /whoami
3. 拿到 xxx@im.wechat
4. 把该 ID 加进对应 project 的 allow_from
5. 重启 cc-connect
```

只有 token 失效、Bot 身份变化、配置丢失时，才需要重新 setup。

## 10. 飞书接入

飞书走 WebSocket 长连接，不需要公网 IP、域名或反向代理。

执行：

```bash
cc-connect feishu setup \
  --config /srv/personal-agent/cc-connect/config.toml \
  --project personal-sandbox
```

如果已经有飞书应用凭证，可以传入：

```bash
cc-connect feishu setup \
  --config /srv/personal-agent/cc-connect/config.toml \
  --project personal-sandbox \
  --app cli_xxx:sec_xxx
```

生成后，飞书平台块类似：

```toml
[[projects.platforms]]
type = "feishu"

[projects.platforms.options]
app_id = "cli_xxxxxxxxxxxxxx"
app_secret = "xxxxxxxxxxxxxxxx"

allow_from = "你的飞书 open_id"
allow_chat = "*"

group_only = false
group_reply_all = false
thread_isolation = true

enable_feishu_card = true
progress_style = "compact"
done_emoji = "Done"
```

我会这样设：

```toml
group_reply_all = false
```

表示群聊里只有 @机器人 才触发，避免监听所有群消息。

```toml
thread_isolation = true
```

表示飞书话题线程尽量隔离成不同会话。

```toml
allow_chat = "*"
```

表示允许任意聊天位置，但仍然要满足 `allow_from`。

如果暂时不用卡片，可以关掉：

```toml
enable_feishu_card = false
progress_style = "legacy"
```

飞书开放平台侧检查：

```text
1. 机器人能力是否启用
2. 应用是否发布
3. 可见范围是否包含目标用户
4. 消息事件是否订阅
5. 是否使用长连接接收事件
```

最小消息事件：

```text
im.message.receive_v1
```

如果使用交互卡片，还需要：

```text
card.action.trigger
```

## 11. 前台验证

先前台启动：

```bash
cc-connect -config /srv/personal-agent/cc-connect/config.toml
```

看到类似日志即可：

```text
level=INFO msg="platform ready" project=personal-sandbox platform=weixin
level=INFO msg="platform ready" project=personal-sandbox platform=feishu
level=INFO msg="engine started" project=personal-sandbox agent=opencode platforms=2
level=INFO msg="cc-connect is running" projects=1
```

按这个顺序验收：

```text
[ ] 直接在 work_dir 运行 OpenCode 正常
[ ] OpenCode 能读取 AGENTS.md
[ ] opencode.jsonc 中的模型可用
[ ] cc-connect 能启动并加载 project
[ ] 微信私聊能触发 personal-sandbox
[ ] 飞书私聊能触发 personal-sandbox
[ ] 飞书群聊只有 @bot 才触发
[ ] allow_from 之外的用户不能触发
[ ] cc-connect 配置中没有重复 model
[ ] Agent 能把附件复制到 inbox/YYYY-MM-DD/
[ ] Agent 能把结果写入 deliverables/<task-name>/
[ ] 高风险命令被 permission 拦住
```

第一个测试 prompt：

```text
请读取 AGENTS.md，总结这个 workspace 的用途和禁止事项。
不要修改文件。
```

第二个测试 prompt：

```text
请在 deliverables/cc-connect-smoke-test/ 下创建一个 README.md，
记录当前测试来自微信还是飞书。
不要读取 workspace 外部目录。
```

这两步跑通后，再交给 systemd。

## 12. systemd 后台运行

先确认 cc-connect 路径：

```bash
command -v cc-connect
```

假设输出是：

```text
/usr/local/bin/cc-connect
```

创建服务：

```bash
sudo tee /etc/systemd/system/cc-connect-agent.service > /dev/null <<'EOF'
[Unit]
Description=cc-connect agent bridge
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=agent
Group=agent
WorkingDirectory=/srv/personal-agent
ExecStart=/usr/local/bin/cc-connect -config /srv/personal-agent/cc-connect/config.toml
Restart=always
RestartSec=5

Environment=HOME=/home/agent

NoNewPrivileges=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
EOF
```

如果 `command -v cc-connect` 不是 `/usr/local/bin/cc-connect`，要修改 `ExecStart`。

启动：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now cc-connect-agent
sudo systemctl status cc-connect-agent
```

看日志：

```bash
journalctl -u cc-connect-agent -f
```

更新配置后重启：

```bash
sudo systemctl restart cc-connect-agent
```

cc-connect 也有自带 daemon。我这里选择 systemd，用固定运行用户、固定 config 路径，和服务器服务管理方式保持一致。

## 13. 多项目 / 多用户

cc-connect 可以一个进程管理多个 project。

推荐原则：

```text
一个用户 / 一个用途 = 一个 project = 一个 work_dir
```

例如给一个量化朋友开独立工作区：

```toml
[[projects]]
name = "quant-friend"
admin_from = "owner_feishu_open_id"

[projects.agent]
type = "opencode"

[projects.agent.options]
work_dir = "/srv/personal-agent/projects/quant-friend"
mode = "default"

[[projects.platforms]]
type = "feishu"

[projects.platforms.options]
app_id = "cli_xxx"
app_secret = "xxx"

allow_from = "friend_feishu_open_id"
allow_chat = "*"

group_only = false
group_reply_all = false
thread_isolation = true

enable_feishu_card = true
progress_style = "compact"
done_emoji = "Done"
```

这样可以做到：

```text
你发消息
  → personal-sandbox
  → /srv/personal-agent/projects/sandbox

朋友发消息
  → quant-friend
  → /srv/personal-agent/projects/quant-friend
```

这里仍然只是逻辑隔离。多个 project 如果都由同一个 Linux 用户 `agent` 运行，本质上共享该用户的系统权限。

更强隔离需要继续升级：

```text
不同 Linux 用户
不同 cc-connect 实例
systemd user service
Docker / Podman
独立 volume
网络限制
```

## 14. 同一个飞书应用复用多个 project

飞书可以复用同一个 `app_id / app_secret` 给多个 project。

注意：

> 多个 project 复用同一个飞书应用时，allow_from / allow_chat 要尽量互斥。

错误例子：

```toml
[[projects]]
name = "personal-sandbox"

[projects.platforms.options]
allow_chat = "*"
# allow_from 没写
```

然后又配置：

```toml
[[projects]]
name = "quant-friend"

[projects.platforms.options]
allow_from = "friend_open_id"
allow_chat = "*"
```

这种情况下，朋友的消息可能先被 `personal-sandbox` 匹配，导致路由错乱。

正确方式：

```toml
[[projects]]
name = "personal-sandbox"

[projects.platforms.options]
allow_from = "your_open_id"
allow_chat = "*"
```

```toml
[[projects]]
name = "quant-friend"

[projects.platforms.options]
allow_from = "friend_open_id"
allow_chat = "*"
```

也就是：

```text
personal-sandbox 只允许你
quant-friend 只允许朋友
```

## 15. 附件处理规则

微信和飞书都能传附件，但 cc-connect 交给 OpenCode 的附件路径可能是临时路径。

规则：后续还会使用的文件，复制到 workspace/inbox/。

写进 `AGENTS.md`：

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

实际对 Agent 下任务时，可以这样说：

```text
请把我刚上传的附件复制到 inbox/2026-06-11/，
保留原始文件名，不要覆盖已有文件。
复制完成后告诉我保存路径、文件大小和 SHA256。
然后再开始分析。
```

公司知识库里的附件通常是原始输入材料，必须进入稳定目录。不要依赖平台临时缓存。

## 16. 微信和飞书的定位

微信适合远程派任务、快速问答、上传小文件和看最终结果。飞书更适合公司入口，群聊 @bot、线程讨论、团队协作、文件沉淀和消息审计都更自然。

复杂任务仍然回到 OpenCode TUI、OpenCode Web UI、IDE 或 SSH。IM 入口只承担派发任务和接收结果。

## 17. 常见坑

### 17.1 cc-connect 里重复写 model

现象：

```text
直接 opencode run 正常
微信 / 飞书触发失败
```

原因：

```toml
model = "xxx"
```

写在了 `[projects.agent.options]` 下，覆盖了 project 的 `opencode.jsonc`。

修复：

```text
cc-connect 不写 model
OpenCode project 的 opencode.jsonc 负责 model
```

### 17.2 只写 admin_from，没有写 allow_from

`admin_from` 只控制管理命令。

真正控制谁能触发 Agent 的是平台层：

```toml
allow_from = "user_id"
```

### 17.3 多 project 复用同一个飞书 app_id，allow_from 太宽

如果一个 project 的飞书平台没有 `allow_from`，它可能抢走其他 project 的消息。

修复：

```text
每个 project 的 allow_from / allow_chat 尽量互斥
```

### 17.4 微信 setup 后没有发第一条消息

微信接入后，需要用户先给 Bot 发一条消息，完成 context_token 缓存。

### 17.5 IM 场景大量使用 ask

`ask` 在 TUI 里很自然，但在微信 / 飞书里容易导致工具调用中止。

修复：

```text
常用安全操作 allow
危险操作 deny
尽量少用 ask
```

### 17.6 附件没有复制到 inbox

平台附件路径可能是临时路径。

修复：

```text
需要长期使用的附件进入 inbox/YYYY-MM-DD/
```

### 17.7 systemd ExecStart 路径错误

npm / release binary / Homebrew 的安装路径可能不同。

启动 systemd 前先确认：

```bash
command -v cc-connect
```

### 17.8 误以为 work_dir 是强隔离

`work_dir` 是 project 级逻辑隔离，不是系统级隔离。

强隔离要靠：

```text
Linux user
container
VM
```

## 18. 公司知识库版本

迁移到公司知识库时，链路不变：

```text
Feishu Group / Private Chat
  ↓
cc-connect
  ↓
OpenCode Project
  ↓
/srv/company-kb
  ↓
AGENTS.md + Skills + opencode.jsonc
```

公司版本要补约束。

### 项目隔离

不同用途拆成不同 project：

```text
company-kb
support-faq
sales-demo
data-analysis
```

### 用户权限

通过：

```text
allow_from
allow_chat
admin_from
```

限制谁能触发、谁能管理、哪个群能用。

### 数据边界

明确：

```text
哪些文件能读
哪些文件能改
哪些内容不能发给外部模型
哪些输出必须人工 review
```

### 交付目录

最终结果进入：

```text
deliverables/<task-name>/
```

重要输出不要散落在聊天记录里。

### 审计

记录：

```text
谁触发
哪个平台
哪个 project
改了哪些文件
生成了哪些交付物
是否经过验证
```

对公司知识库来说，cc-connect 的价值是把团队协作平台里的明确任务送进一个受约束的 OpenCode workspace。

## 19. 小结

这次实践后的结论：

1. cc-connect 做消息入口和 project 路由。
2. OpenCode 执行任务，模型和权限放在 project 的 `opencode.jsonc`。
3. `work_dir`、`allow_from`、`admin_from` 是配置重点。
4. 飞书群聊建议 `group_reply_all = false`。
5. IM 场景减少 `ask`，权限提前设计成 allow / deny。
6. 附件进 `inbox/`，输出进 `deliverables/`。
7. 多项目是逻辑隔离，强隔离要靠系统层设计。

核心方案：

> 用 cc-connect 做微信 / 飞书到 OpenCode project 的路由层；
> 用 OpenCode project 自己的 AGENTS.md、Skills、opencode.jsonc 管住执行边界；
> 用 workspace 目录结构沉淀输入、过程和交付物。

这是公司 KnowledgeBase Agent 继续往前走时需要的一层：让 Agent 进入团队真实使用的协作入口。
