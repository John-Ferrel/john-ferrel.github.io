---
title: Building KnowledgeBase with OpenCode 2 - Docker and SFTPGo
tags:
  - ai-coding
  - opencode
  - knowledge-base
  - saas-copilot
  - docker
  - sftpgo
  - rag
draft: false
created: 2026-06-10 00:41
modified: 2026-06-16 18:59
---
上一篇[[Building KnowledgeBase with Opencode]]我记录了一个比较轻量的知识库冷启动方案：用 OpenCode Web 作为知识库草稿整理工作台，用 Git 管理知识源文件，用 AGENTS.md / skills / commands / permissions 约束格式、流程和安全边界。

下一篇：[[Building KnowledgeBase with OpenCode 3 - CC Connect]]。

当时的核心链路是：

```text
同事通过浏览器访问 OpenCode Web
        ↓
粘贴业务说明、会议纪要、页面截图、指标解释
        ↓
OpenCode 按规则整理成 Markdown 草稿
        ↓
草稿进入 review/pending/
        ↓
人工审核后进入 knowledge/ 或 review/approved/
        ↓
后续导入 Dify / 向量库 / SaaS Copilot
```

这个方案适合作为第一版 POC，但继续往前走时，我遇到了两个新的问题：

```text
1. OpenCode Web 本身不是文件上传系统
2. OpenCode Web 的权限范围接近运行它的 Linux 用户
```

也就是说，如果我只是把 OpenCode Web 直接跑在服务器上，业务同事确实可以通过浏览器整理知识库草稿，但他们没有一个自然的入口把 raw files 放进 workspace；另一方面，如果 OpenCode Web 是用宿主机上的普通用户甚至 root 启动的，它就不再是一个“项目级 agent”，而更接近一个“服务器级 agent”。

所以这篇是上一篇的补充：**用 Docker 封装 OpenCode Web，并用 SFTPGo WebClient 提供文件上传入口。**

最终目标是把它升级成：

```text
OpenCode Web + SFTPGo WebClient 的项目级知识库 agent 工作区
```

---

## 1. 这次升级解决什么问题

第一版的 OpenCode Web 方案主要解决：

```text
知识整理
草稿生成
规则约束
人工 review
```

但它没有很好解决：

```text
raw files 如何进入 workspace
OpenCode Web 如何避免访问整个服务器
如何让这个 agent 只服务于 company-kb 项目
```

所以这次升级的目标是：

```text
1. 给同事一个浏览器文件上传入口
2. 把上传文件统一放到 inbox/uploaded-docs/
3. 让 OpenCode Web 只看到项目目录
4. 保持原来的 AGENTS.md / skills / commands / Git review 结构不变
5. 尽量小改动，不重做整套系统
```

---

## 2. 新架构

升级后，结构变成：

```text
业务同事
  ├── SFTPGo WebClient
  │       ↓
  │   上传 raw files
  │       ↓
  │   /srv/company-kb/inbox/uploaded-docs/
  │
  └── OpenCode Web
          ↓
      读取 inbox/uploaded-docs/
          ↓
      生成 review/pending/*.md
          ↓
      人工 review
          ↓
      knowledge/ 或 review/approved/
```

容器视角是：

```text
Host:
  /srv/company-kb/

OpenCode container:
  /workspace  ->  /srv/company-kb/

SFTPGo container:
  upload dir  ->  /srv/company-kb/inbox/uploaded-docs/
```

这个变化很关键。

之前 OpenCode Web 直接运行在宿主机上，它理论上可能接触到服务器上的更多路径。现在它运行在 Docker 容器里，并且只挂载 `/srv/company-kb` 到容器内的 `/workspace`。

也就是说，这个 agent 的工作范围变成：

```text
/workspace
```

而不是：

```text
整台服务器
```

这更接近我想要的“项目级 agent”。

---

## 3. 保留原来的 company-kb repo

这次升级不需要重建知识库 repo。

继续使用：

```text
/srv/company-kb/
```

原来的结构继续保留：

```text
company-kb/
├── AGENTS.md
├── opencode.jsonc
├── README.md
├── START_HERE.md
├── inbox/
│   └── uploaded-docs/
├── knowledge/
├── review/
│   └── pending/
├── scripts/
│   └── validate_kb.py
└── .opencode/
    ├── commands/
    │   └── kb-start.md
    └── skills/
```

只是把运行方式从宿主机 systemd 改成 Docker Compose。

---

## 4. 停掉旧的 systemd 服务

如果之前是用 systemd 托管 OpenCode Web，可以先停掉。

例如服务名是：

```text
opencode-kb-web.service
```

执行：

```bash
sudo systemctl stop opencode-kb-web
sudo systemctl disable opencode-kb-web
```

不建议立刻删除 service 文件。这样如果 Docker 方案出问题，还可以快速回滚：

```bash
cd /opt/kb-agent-stack
sudo docker compose down

sudo systemctl enable --now opencode-kb-web
```

---

## 5. 准备 Docker Compose 目录

新建一个独立的 stack 目录：

```bash
sudo mkdir -p /opt/kb-agent-stack/opencode
sudo mkdir -p /opt/kb-agent-stack/opencode-home
sudo mkdir -p /opt/kb-agent-stack/sftpgo-data
sudo mkdir -p /srv/company-kb/inbox/uploaded-docs
```

这里几个目录的作用是：

```text
/opt/kb-agent-stack/
  存放 Docker Compose 和容器相关配置

/opt/kb-agent-stack/opencode-home/
  持久化 OpenCode 容器里的 home，包括 auth 等状态

/opt/kb-agent-stack/sftpgo-data/
  持久化 SFTPGo 数据

/srv/company-kb/
  真实知识库 repo

/srv/company-kb/inbox/uploaded-docs/
  SFTPGo 上传文件落地目录
```

---

## 6. OpenCode Dockerfile

我这里没有使用一个复杂的自定义镜像，只做一个很薄的 Node 镜像，然后安装 opencode。

创建：

```bash
nano /opt/kb-agent-stack/opencode/Dockerfile
```

写入：

```dockerfile
FROM node:22-bookworm-slim

RUN apt-get update && apt-get install -y \
    bash \
    curl \
    git \
    ca-certificates \
    python3 \
    jq \
  && rm -rf /var/lib/apt/lists/*

RUN npm install -g opencode-ai

RUN groupadd -g 10001 opencode \
  && useradd -m -u 10001 -g 10001 -s /bin/bash opencode \
  && mkdir -p /workspace \
  && chown -R opencode:opencode /workspace /home/opencode

USER opencode
ENV HOME=/home/opencode
WORKDIR /workspace

EXPOSE 4096

CMD ["opencode", "web", "--hostname", "0.0.0.0", "--port", "4096"]
```

几个要点：

```text
1. OpenCode 在容器里运行
2. 默认工作目录是 /workspace
3. 运行用户不是 root
4. /workspace 会挂载到宿主机 /srv/company-kb
```

---

## 7. OpenCode 环境变量

创建：

```bash
nano /opt/kb-agent-stack/.env.opencode
```

写入：

```bash
OPENCODE_SERVER_USERNAME=opencode
OPENCODE_SERVER_PASSWORD=change-this-to-a-strong-password
```

权限收紧：

```bash
sudo chmod 600 /opt/kb-agent-stack/.env.opencode
```

模型凭据有两种做法：

```text
1. 在容器里重新执行 opencode auth login / /connect
2. 通过环境变量提供 provider API key
```

我最后采用的是在容器中重新完成 OpenCode 登录配置。因为 `/home/opencode` 已经被挂载到宿主机的 `opencode-home/`，所以容器重启后认证状态不会丢。

---

## 8. docker-compose.yml

创建：

```bash
nano /opt/kb-agent-stack/docker-compose.yml
```

写入：

```yaml
services:
  opencode-web:
    build:
      context: ./opencode
    container_name: kb-opencode-web
    restart: unless-stopped

    working_dir: /workspace

    env_file:
      - ./.env.opencode

    environment:
      HOME: /home/opencode

    ports:
      - "4096:4096"

    volumes:
      - /srv/company-kb:/workspace
      - ./opencode-home:/home/opencode

    security_opt:
      - no-new-privileges:true

    cap_drop:
      - ALL

  sftpgo:
    image: drakkan/sftpgo:latest
    container_name: kb-sftpgo
    restart: unless-stopped

    ports:
      - "8080:8080"

    volumes:
      - ./sftpgo-data:/var/lib/sftpgo
      - /srv/company-kb/inbox/uploaded-docs:/srv/sftpgo/data/uploads

    environment:
      SFTPGO_HTTPD__BINDINGS__0__PORT: "8080"
      SFTPGO_HOOK__DISABLE_DOT_ENTRIES: "1"

    security_opt:
      - no-new-privileges:true
```


SFTPGo 的职责非常窄：

```text
只负责 WebClient 上传文件
不负责知识整理
不负责 RAG
不负责审核
```

---

## 9. 启动服务

执行：

```bash
cd /opt/kb-agent-stack

sudo docker compose build
sudo docker compose up -d
```

检查：

```bash
sudo docker compose ps
sudo docker compose logs -f opencode-web
sudo docker compose logs -f sftpgo
```

访问：

```text
OpenCode Web:
http://SERVER_IP:4096

SFTPGo:
http://SERVER_IP:8080
```

SFTPGo 管理入口一般是：

```text
http://SERVER_IP:8080/web/admin
```

SFTPGo 普通用户 WebClient 入口一般是：

```text
http://SERVER_IP:8080/web/client
```

---

## 10. 配置 SFTPGo WebClient

第一次进入 SFTPGo admin 后，创建 admin 用户。

然后创建一个普通上传用户，例如：

```text
username: kb-uploader
home dir: /srv/sftpgo/data/uploads
```

这个 home dir 是容器内路径，实际映射到宿主机：

```text
/srv/company-kb/inbox/uploaded-docs/
```

所以同事通过 SFTPGo WebClient 上传文件后，宿主机可以看到：

```bash
ls -la /srv/company-kb/inbox/uploaded-docs
```

OpenCode 容器内也能看到：

```bash
ls -la /workspace/inbox/uploaded-docs
```

这条链路就是：

```text
SFTPGo WebClient
        ↓
/srv/company-kb/inbox/uploaded-docs
        ↓
OpenCode container: /workspace/inbox/uploaded-docs
```

---

## 11. 配置 OpenCode 模型

进入 OpenCode 容器：

```bash
cd /opt/kb-agent-stack
sudo docker compose exec opencode-web bash
```

容器内确认当前目录：

```bash
cd /workspace
pwd
```

应该是：

```text
/workspace
```

然后配置 OpenCode：

```bash
opencode auth login
```

或者进入 TUI / Web 后执行：

```text
/connect
/models
```

这里有一个重要细节：**Docker 方案下，OpenCode Web 中应该选择 `/workspace` 作为项目工作区，而不是宿主机路径 `/srv/company-kb`。**

宿主机上的：

```text
/srv/company-kb
```

只是 Docker bind mount 的来源路径。

容器里的项目路径才是：

```text
/workspace
```

如果在 OpenCode Web 里还打开旧路径 `/srv/company-kb`，就可能出现：

```text
AGENTS.md 没加载
.opencode/commands 没发现
/kb-start 不出现
skills 行为不稳定
```

这不是 commands 或 skills 本身的问题，而是当前 session 打开的 workspace 不对。

---

## 12. 验证 workspace

进入容器：

```bash
cd /opt/kb-agent-stack
sudo docker compose exec opencode-web bash
```

检查：

```bash
cd /workspace
pwd
git status
ls -la .opencode
ls -la .opencode/commands
cat .opencode/commands/kb-start.md
```

理想结果：

```text
/workspace
.git 存在
.opencode/commands/kb-start.md 存在
AGENTS.md 存在
START_HERE.md 存在
opencode.jsonc 存在
```

然后在 OpenCode Web 里：

```text
1. 选择 /workspace 作为 project
2. 新建 session
3. 输入 /kb-start
```

如果 `/kb-start` 仍然不出现，先不要急着改配置。优先确认当前 Web session 是否真的是 `/workspace`。

---

## 13. Git dubious ownership 问题

容器化之后，可能会遇到宿主机运行 Git 时的错误：

```text
fatal: detected dubious ownership in repository at '/srv/company-kb'
```

这是因为 `/srv/company-kb` 的 owner 可能变成了容器用户，宿主机当前用户和 repo owner 不一致。

最小处理方式是：

```bash
git config --global --add safe.directory /srv/company-kb
```

然后再试：

```bash
cd /srv/company-kb
git status
```

这不是 Git repo 丢了，也不是 Docker 挂载不一致，而是 Git 的安全机制。

不过这样, Git 不能操作的问题还是没有解决.

只能通过

```bash
cd /opt/kb-agent-stack
sudo docker compose exec opencode-web bash
```

进入容器环境进行 add/commit 等.

---

## 14. 更新 AGENTS.md / README.md / START_HERE.md

引入 SFTPGo 后，项目说明文件也要更新。

否则同事会以为所有东西都从 OpenCode Web 进入，实际上现在有两个入口：

```text
SFTPGo WebClient：上传 raw files
OpenCode Web：整理知识库草稿
```

### AGENTS.md 增加

```md
## Raw File Ingestion

Raw files uploaded by business users are placed under:

- `inbox/uploaded-docs/`

These files are uploaded through SFTPGo WebClient, not through OpenCode Web.

When the user asks you to process uploaded files, first check the file path under:

- `inbox/uploaded-docs/`

Then create structured draft documents under:

- `review/pending/`

## Supported Input Materials

Preferred input materials:

- Markdown
- TXT
- CSV
- JSON
- YAML / YML
- page screenshots
- chart screenshots
- process screenshots
- de-identified business notes
- de-identified meeting notes

For office documents such as PDF, DOCX, XLSX, PPTX, scanned documents, image-based PDFs, zip files, or large attachments:

- Do not assume they can be directly parsed reliably.
- Ask the user to convert them to Markdown, TXT, CSV, JSON, YAML, or screenshots first.
- If the file is present but cannot be read as text, explain that preprocessing is required.

Uploaded files should be treated as raw materials, not approved knowledge.
```

### START_HERE.md 增加

````md
# OpenCode Web 知识库工作台使用说明

这个工作台由两个入口组成：

```text
SFTPGo WebClient：上传原始文件
OpenCode Web：整理知识库草稿
````

上传文件请使用：

```text
http://SERVER_IP:8080/web/client
```

上传后的文件会进入：

```text
inbox/uploaded-docs/
```

整理知识库请使用：

```text
http://SERVER_IP:4096
```

第一次使用 OpenCode Web，请输入：

```text
/kb-start
```

所有新草稿请保存到：

```text
review/pending/
```

````

### kb-start command 也需要更新

`.opencode/commands/kb-start.md` 应该提醒 agent 说明两个入口：

```md
The message should include:

1. What this workspace is for
2. The two entry points: SFTPGo WebClient for upload, OpenCode Web for authoring
3. What kinds of documents the user can create
4. Where uploaded files are stored: `inbox/uploaded-docs/`
5. Where drafts should be saved: `review/pending/`
6. What the user must not upload or ask you to do
7. Three recommended example prompts
8. A reminder that uncertain information should go into Open Questions
````

---

## 15. 输入文件范围

因为 OpenCode Web 不是完整文档解析管线，所以需要告诉同事“哪些文件适合直接使用”。

建议支持：

```text
Markdown
TXT
CSV
JSON
YAML / YML
页面截图
图表截图
流程截图
已脱敏业务说明
已脱敏会议纪要
```

暂不建议直接处理：

```text
PDF
DOCX
XLSX
PPTX
扫描件
图片版 PDF
压缩包
大型附件
复杂多 sheet 表格
```

这些文件最好先转换成：

```text
Markdown
TXT
CSV
JSON
YAML
截图
```

再交给 OpenCode 整理。

我测试上传了一个 `.xlsx` 文件, 可通过zip解压, 或python pandas等来读取, 但这不代表它总是能可靠解析 Excel。对这种文件，最好先导出成 CSV 或 Markdown。

---

## 16. 端到端验收

这次升级完成后，我的验收标准是：

```text
[ ] OpenCode Web 可以访问
[ ] SFTPGo WebClient 可以访问
[ ] SFTPGo 上传文件能落到 /srv/company-kb/inbox/uploaded-docs
[ ] OpenCode 容器内能看到 /workspace/inbox/uploaded-docs
[ ] OpenCode Web 使用的是 /workspace project
[ ] /kb-start 能被发现
[ ] AGENTS.md 生效
[ ] skills 可以按任务被使用
[ ] OpenCode 能读取 uploaded-docs 中的文本文件
[ ] OpenCode 能生成 review/pending/*.md
[ ] git diff 能看到草稿变更
[ ] validate_kb.py 能跑
```

建议先用 `.md` 或 `.txt` 做测试，不要一开始用 `.xlsx`。

例如创建测试文件：

```bash
cat > /srv/company-kb/inbox/uploaded-docs/test-note.md <<'EOF'
Sell-through means the ratio of sold units to ordered units.
The exact denominator needs business confirmation.
EOF
```

然后在 OpenCode Web 里问：

```text
请读取 inbox/uploaded-docs/test-note.md，并整理成 metric_definition 草稿，保存到 review/pending/test-sell-through.md。公式不确定处写 Needs confirmation。
```

检查：

```bash
cd /srv/company-kb
git diff
ls -la review/pending
python3 scripts/validate_kb.py
```

如果这一步成功，说明核心流程已经跑通。

---

## 17. 当前方案的边界

升级后，这套方案更安全、更完整，但它仍然不是正式企业知识库系统。

它适合：

```text
内部 POC
小范围同事试用
知识库冷启动
业务文档草稿整理
FAQ / 指标定义 / 页面说明沉淀
```

它不适合直接承担：

```text
正式多人协同文档系统
复杂权限审批
生产级文档解析管线
大规模文件处理
多租户 SaaS 后台
```

也就是说，SFTPGo 和 Docker 解决的是：

```text
文件上传入口
项目级隔离
运行边界
```

但不解决：

```text
知识审核
内容准确性
RAG 切块质量
最终 Copilot 回答质量
```

这些仍然要靠 Git review、validate 脚本、Dify/向量库处理和后续评测集。

---

## 18. 总结

上一篇的方案是：

```text
OpenCode Web + Git repo + AGENTS.md + skills + permissions
```

这次升级后变成：

```text
Docker Compose
  ├── OpenCode Web
  │     └── /workspace -> /srv/company-kb
  │
  └── SFTPGo WebClient
        └── uploads -> /srv/company-kb/inbox/uploaded-docs
```

我觉得这个版本更接近一个真正可用的内部 POC：

```text
SFTPGo 解决 raw file upload
OpenCode Web 解决知识整理
Docker 解决项目级隔离
AGENTS.md / skills / commands 解决工作流约束
Git review 解决知识治理
```

主要解决的问题是是：

> 如果要给同事使用，OpenCode Web 不应该裸跑在宿主机上。  
> 它更适合作为一个被 Docker 限制在 `/workspace` 内的项目级 agent。
