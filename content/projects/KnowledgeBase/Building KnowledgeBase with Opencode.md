---
title: Building KnowledgeBase with Opencode
tags:
  - AICoding
  - OpenCode
  - KnowledgeBase
  - SaaSCopilot
  - RAG
draft: false
created: 2026-06-08 23:18
modified: 2026-06-08 23:43
---
最近我在做公司的 SaaS Copilot, 需要设计知识库。我重新思考企业知识库和 SaaS Copilot 的关系。

最初的想法比较传统：做一个协同文档系统，接文档处理管线，最后导入 Dify 或向量数据库，供 agents / workflow 调用。我对这个想法很满意, 但是开发周期比较长, 尤其是如果要考虑业务同事的使用体验和冷启动, 完整开发一套“协同云文档 + 审核流 + RAG 后台”可能不太能被接受。

于是我想到一个更轻的方案：

> 用 OpenCode Web 作为知识库内容生产工作台，让业务同事通过浏览器整理文档；用 Git 管理知识源文件；用 AGENTS.md / skills / commands / permissions 约束格式、流程和安全边界；后续再把 approved 文档导入 Dify 或向量库

这篇记录主要写部署方案。

---

## 1. 目标

当前目标不是做一个完整的企业知识库系统，而是先搭一个内部 POC：

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

它的定位是：

```text
OpenCode Web = 知识库草稿整理工作台
Git repo     = 知识资产主账本
Dify/RAG     = 知识服务层
Java/Python  = SaaS Copilot 产品控制层
```

所以，OpenCode Web 不是正式知识库门户(因为呈现没有严谨的文档结构)，也不是多人协同文档系统(也不是成熟的多人协同文档系统)。它更像是一个 agent-assisted authoring workspace。

---

## 2. 整体架构

我希望最终形成这样的结构：

```text
知识生产侧：

业务同事
   ↓
OpenCode Web
   ↓
company-kb Git repo
   ↓
review / approve
   ↓
export_to_dify / chunking / embedding
   ↓
Dify / Vector DB


Copilot调用侧：

Java SaaS 主系统
   ↓
Python Adapter
   ↓
Dify / Vector DB / OpenCode runtime
   ↓
Python Adapter 统一 response schema
   ↓
Java UI 展示
```

目前只部署第一段：

```text
OpenCode Web
   ↓
company-kb repo
   ↓
人工 review
```

暂时不做：

```text
opencode serve
Python Adapter 调 OpenCode
Dify 调 OpenCode
OpenCode 调 Dify
```

这可以避免一开始架构过度复杂。

---

## 3. 服务器目录规划

服务器上准备这样放：

```text
/srv/company-kb/
  企业知识库 repo

/etc/company-kb/
  OpenCode Web 环境变量，比如登录密码

/etc/systemd/system/opencode-kb-web.service
  systemd 服务
```

知识库 repo 结构：

```text
company-kb/
├── AGENTS.md
├── opencode.jsonc
├── README.md
├── inbox/
│   ├── raw-notes/
│   ├── screenshots/
│   ├── uploaded-docs/
│   └── meeting-notes/
├── knowledge/
│   ├── common/
│   └── mp/
│       ├── metrics/
│       ├── page-guides/
│       ├── business-rules/
│       ├── sop/
│       └── faq/
├── glossary/
├── review/
│   ├── pending/
│   ├── approved/
│   └── rejected/
├── schemas/
├── scripts/
│   └── validate_kb.py
├── outputs/
│   ├── dify_import/
│   └── vector_chunks/
└── .opencode/
    └── skills/
        ├── create-metric-definition/
        ├── create-faq/
        └── kb-quality-review/
```

核心思路：

```text
inbox/          放原始材料
review/pending/ 放草稿
knowledge/      放正式知识文档
scripts/        放校验和导出脚本
.opencode/      放 OpenCode skills
```

---

## 4. 安装 OpenCode

SSH 登录服务器后，先安装基础工具：

```bash
sudo apt update
sudo apt install -y curl git ca-certificates build-essential jq python3 python3-venv
```

安装 OpenCode：

```bash
curl -fsSL https://opencode.ai/install | bash
```

刷新 shell：

```bash
exec $SHELL -l
```

检查：

```bash
which opencode
opencode --version
```

如果 `which opencode` 找不到，需要检查安装路径是否已经加入 `PATH`。

---

## 5. 创建知识库 repo

```bash
sudo mkdir -p /srv/company-kb
sudo chown -R "$USER":"$USER" /srv/company-kb

cd /srv/company-kb
git init
```

创建目录：

```bash
mkdir -p \
  inbox/raw-notes \
  inbox/screenshots \
  inbox/uploaded-docs \
  inbox/meeting-notes \
  knowledge/common \
  knowledge/mp/metrics \
  knowledge/mp/page-guides \
  knowledge/mp/business-rules \
  knowledge/mp/sop \
  knowledge/mp/faq \
  glossary \
  review/pending \
  review/approved \
  review/rejected \
  schemas \
  scripts \
  outputs/dify_import \
  outputs/vector_chunks \
  .opencode/skills/create-metric-definition \
  .opencode/skills/create-faq \
  .opencode/skills/kb-quality-review
```

写一个简单 README：

```bash
cat > README.md <<'EOF'
# Company Knowledge Base

This repository is the source of truth for the internal SaaS Copilot knowledge base.

Workflow:

1. Raw materials go to `inbox/`
2. Draft knowledge documents go to `review/pending/`
3. Approved documents go to `review/approved/` or `knowledge/`
4. Approved documents can later be exported to Dify / vector database

OpenCode Web is used as an assisted authoring workspace, not as the final knowledge portal.
EOF
```

---

## 6. 写 AGENTS.md

`AGENTS.md` 是这个方案的核心。它把 OpenCode 从“代码助手”约束成“知识库整理助手”。

```bash
cat > AGENTS.md <<'EOF'
# Company Knowledge Base Authoring Agent

You are helping business users create a structured knowledge base for an internal SaaS Copilot.

## Main Role

You are a knowledge base authoring assistant.

You help users:
- organize raw notes into structured Markdown documents
- extract FAQ items
- define business metrics
- normalize terminology
- convert screenshots or pasted page content into documentation
- identify missing information
- prepare draft documents for review

## Important Boundary

This repository is a knowledge source for a SaaS Copilot.

You must not:
- invent business facts
- silently fill missing formulas
- modify system configuration unless explicitly asked by the project owner
- run destructive shell commands
- move draft documents to approved status without human approval
- expose secrets, credentials, tokens, or environment files

## Directory Rules

Raw input should go under:

- inbox/raw-notes/
- inbox/screenshots/
- inbox/uploaded-docs/
- inbox/meeting-notes/

Draft knowledge documents should go under:

- review/pending/

Approved documents should go under:

- review/approved/

Do not directly create approved documents unless explicitly instructed by the project owner.

## Required Frontmatter

Every knowledge document must include:

---
id:
title:
domain:
module:
doc_type:
owner:
source:
status: draft
confidence: low | medium | high
last_updated:
tags:
---

## Allowed doc_type

Use one of:

- metric_definition
- business_rule
- page_guide
- sop
- faq
- troubleshooting
- glossary
- case_note
- data_boundary

## Writing Rules

- Prefer concise Markdown.
- Separate confirmed facts from assumptions.
- Put uncertain items under "Open Questions".
- If a formula is unclear, mark it as "Needs confirmation".
- If source quality is weak, set confidence: low.
- Do not remove caveats.
- Do not over-polish business uncertainty into fake certainty.
- Use business-friendly language.

## Output Quality Checklist

Before finishing a document, check:

- Does it have required frontmatter?
- Is the owner/source clear?
- Are formulas and definitions explicit?
- Are caveats included?
- Are open questions listed?
- Is the document suitable for RAG retrieval?
EOF
```

---

## 7. 写 opencode.jsonc

这里主要做三件事：

1. 指定 server port 和 hostname；
2. 加载 `AGENTS.md` 和 `README.md`；
3. 收紧权限。

```bash
cat > opencode.jsonc <<'EOF'
{
  "$schema": "https://opencode.ai/config.json",

  "autoupdate": false,

  "server": {
    "port": 4096,
    "hostname": "0.0.0.0"
  },

  "instructions": [
    "AGENTS.md",
    "README.md"
  ],

  "permission": {
    "*": "ask",

    "read": {
      "*": "allow",
      "*.env": "deny",
      "*.env.*": "deny",
      "/etc/*": "deny"
    },

    "bash": {
      "*": "ask",
      "git status": "allow",
      "git status *": "allow",
      "git diff": "allow",
      "git diff *": "allow",
      "git log": "allow",
      "git log *": "allow",
      "python scripts/validate_kb.py": "allow",

      "rm *": "deny",
      "rm -rf *": "deny",
      "sudo *": "deny",
      "curl *": "deny",
      "wget *": "deny",
      "chmod *": "deny",
      "chown *": "deny",
      "systemctl *": "deny"
    },

    "edit": {
      "*": "deny",

      "inbox/*.md": "allow",
      "inbox/raw-notes/*.md": "allow",
      "inbox/meeting-notes/*.md": "allow",
      "review/pending/*.md": "allow",

      "knowledge/*.md": "ask",
      "knowledge/common/*.md": "ask",
      "knowledge/mp/metrics/*.md": "ask",
      "knowledge/mp/page-guides/*.md": "ask",
      "knowledge/mp/business-rules/*.md": "ask",
      "knowledge/mp/sop/*.md": "ask",
      "knowledge/mp/faq/*.md": "ask",
      "glossary/*.md": "ask"
    },

    "webfetch": "deny",
    "websearch": "deny",
    "external_directory": "deny"
  }
}
EOF
```

这个配置的核心原则是：

```text
允许读知识库；
允许写 inbox/ 和 review/pending/；
允许看 git diff；
允许跑 validate_kb.py；
禁止 rm、sudo、curl、wget、systemctl 等危险命令。
```

如果后面需要进一步治理模型，可以把模型和 provider 限制放在 `/etc/opencode/opencode.jsonc` 里，例如：

```jsonc
{
  "$schema": "https://opencode.ai/config.json",

  "enabled_providers": ["openai"],

  "model": "openai/gpt-5.1",
  "small_model": "openai/gpt-5.1",

  "autoupdate": false
}
```

这里的模型名要以实际 `opencode models` 输出为准。

---

## 8. 用 skills 和 commands 规范轻量工作流

一开始我只是把 skills 理解成“额外提示词”或“可复用说明”。但在这个知识库工作台场景里，skills 的价值更明确：

> skills 可以用来规范重复出现的知识整理工作流。

比如同事经常会做这些任务：

```text
整理指标定义
整理 FAQ
整理页面说明
整理会议纪要
审查知识库草稿
```

这些任务每次都要求同事写完整 prompt 并不现实。更好的方式是把它们沉淀为 skills。

我的分工是：

```text
AGENTS.md
  规定整个 repo 的长期规则、目录边界、文档格式和安全原则。

skills
  规定某一类任务应该如何完成，比如如何整理指标定义、如何生成 FAQ、如何审查文档。

commands
  给同事提供可直接触发的入口，比如 /kb-start、/create-faq、/review-kb。
```

这样，OpenCode Web 就不只是一个聊天窗口，而是一个被规则和工作流约束过的知识库工作台。

---

### 8.1 skills 目录

第一版先放三个 skills：

```text
.opencode/skills/
├── create-metric-definition/
│   └── SKILL.md
├── create-faq/
│   └── SKILL.md
└── kb-quality-review/
    └── SKILL.md
```

这三个已经足够覆盖早期知识库冷启动：

```text
create-metric-definition
  把业务同事对指标的解释整理成结构化指标定义。

create-faq
  把业务问答、支持问题、会议讨论整理成 FAQ。

kb-quality-review
  检查知识库草稿是否适合进入正式知识库。
```

---

### 8.2 指标定义整理 skill

```bash
cat > .opencode/skills/create-metric-definition/SKILL.md <<'EOF'
---
name: create-metric-definition
description: Use this when converting business explanations into a structured metric definition document for the SaaS Copilot knowledge base.
---

# Metric Definition Skill

Create a Markdown document with:

- frontmatter
- definition
- formula
- business meaning
- example interpretation
- common misunderstandings
- caveats
- related questions
- open questions

Never invent a formula. If the formula is not provided, write "Needs confirmation".

The document must be saved as status: draft.

Uncertain content must be placed under "Open Questions".
EOF
```

这个 skill 的重点是：不让 agent 自己脑补公式。

对于企业知识库来说，公式、指标口径、业务规则这类内容最怕“写得像真的”。所以我宁可让它明确写 `Needs confirmation`，也不要让它生成一个看起来很顺的错误定义。

---

### 8.3 FAQ 整理 skill

```bash
cat > .opencode/skills/create-faq/SKILL.md <<'EOF'
---
name: create-faq
description: Use this when turning business notes, user questions, or support discussions into structured FAQ documents.
---

# FAQ Skill

Extract question-answer pairs.

Each FAQ item should include:

- question
- short answer
- detailed answer
- related module
- tags
- caveats
- source

Mark uncertain answers as low confidence.

Do not invent business facts.
If the source is unclear, put the issue under "Open Questions".
EOF
```

FAQ 是知识库冷启动里性价比最高的一类内容。因为业务同事通常不一定愿意写完整文档，但他们经常能提供“客户常问什么”“这个页面怎么解释”“这个指标一般怎么理解”。

把这些内容转成 FAQ，很适合作为 SaaS Copilot 的早期知识来源。

---

### 8.4 知识质量审查 skill

```bash
cat > .opencode/skills/kb-quality-review/SKILL.md <<'EOF'
---
name: kb-quality-review
description: Use this when reviewing knowledge base documents for completeness, uncertainty, RAG suitability, and business correctness risks.
---

# KB Quality Review Skill

Review the document for:

- missing frontmatter
- unclear source
- unsupported claims
- missing caveats
- ambiguous formulas
- duplicated concepts
- poor retrieval title
- overly long sections
- inconsistent terminology
- content that should be moved to Open Questions

Do not silently fix business facts.

Return the review in this structure:

## Summary

## Blocking Issues

## Suggested Improvements

## Open Questions

## RAG Suitability
EOF
```

这个 skill 的作用是把 review 也流程化。

这样我不需要每次都写：

```text
请检查 frontmatter、来源、caveats、RAG 友好性、不确定内容……
```

只需要让 agent 使用 `kb-quality-review` 的规则即可。

---

### 8.5 增加 /kb-start 开场命令

除了 skills，我还会加一个开场命令，降低同事第一次使用的门槛。

目录：

```text
.opencode/commands/
└── kb-start.md
```

创建命令：

```bash
mkdir -p .opencode/commands

cat > .opencode/commands/kb-start.md <<'EOF'
---
description: Show the onboarding guide for the knowledge base authoring workspace
agent: plan
---

Please read `START_HERE.md` and `AGENTS.md`.

Then give the user a concise onboarding message in Chinese.

The message should include:

1. What this workspace is for
2. What kinds of documents the user can create
3. Where drafts should be saved
4. What the user must not upload or ask you to do
5. Three recommended example prompts
6. A reminder that uncertain information should go into Open Questions

Do not modify any files.
Do not run shell commands.
Keep the response friendly, practical, and under 500 Chinese characters.
EOF
```

然后创建 `START_HERE.md`：

```bash
cat > START_HERE.md <<'EOF'
# OpenCode Web 知识库工作台使用说明

欢迎使用内部知识库草稿整理工作台。

这个工作台用于帮助你把业务说明、指标定义、FAQ、会议纪要、页面截图说明等内容整理成结构化 Markdown 草稿。

## 你可以做什么

你可以让 agent 帮你：

- 整理指标定义
- 整理 FAQ
- 整理页面说明
- 整理业务规则
- 整理 SOP / 操作流程
- 审查知识库草稿
- 把会议纪要转成知识库文档
- 把不确定内容归入 Open Questions

## 草稿保存位置

所有草稿请保存到：

review/pending/

不要直接保存到：

review/approved/

## 请不要做什么

请不要上传或粘贴：

- 密码
- API key
- token
- 数据库连接串
- 未脱敏客户隐私数据
- 生产系统配置
- 未授权的敏感文件

请不要要求 agent：

- 删除文件
- 执行 sudo
- 修改系统配置
- 直接把草稿标记为 approved
- 编造公式、指标口径或业务事实

## 推荐提问

请把下面内容整理成 MP 模块的 FAQ 文档，保存到 review/pending/xxx.md。

请把这段指标说明整理成 metric_definition 文档，公式不确定处标记 Needs confirmation。

请审查 review/pending/xxx.md 是否适合进入知识库。
EOF
```

之后同事第一次进入 OpenCode Web，只需要输入：

```text
/kb-start
```

这比让他们自己读 README 或 AGENTS.md 友好很多。

---

### 8.6 小结

最后，提交这些文件：

```bash
git add START_HERE.md .opencode/skills .opencode/commands
git commit -m "Add KB authoring skills and onboarding command"
```

这一层设计的价值是：

```text
AGENTS.md 保证长期边界；
skills 规范重复工作流；
commands 降低同事使用门槛；
permissions 控制安全风险；
Git review 保证知识质量。
```

因为我们本质上是在用一个通用 agent，承担 project 级别的知识库整理角色，所以需要用 AGENTS.md、skills、commands 和 permissions 给它建立一个被轻量流程约束过的知识生产环境。


---

## 9. 增加 validate_kb.py

为了避免知识库格式完全失控，先加一个最小校验脚本。

```bash
cat > scripts/validate_kb.py <<'EOF'
#!/usr/bin/env python3
from pathlib import Path
import re
import sys

REQUIRED_FIELDS = [
    "id",
    "title",
    "domain",
    "module",
    "doc_type",
    "owner",
    "source",
    "status",
    "confidence",
    "last_updated",
    "tags",
]

VALID_DOC_TYPES = {
    "metric_definition",
    "business_rule",
    "page_guide",
    "sop",
    "faq",
    "troubleshooting",
    "glossary",
    "case_note",
    "data_boundary",
}

VALID_CONFIDENCE = {"low", "medium", "high"}
VALID_STATUS = {"draft", "approved", "rejected"}

DOCS = list(Path("review/pending").glob("*.md")) + list(Path("knowledge").glob("**/*.md"))

def parse_frontmatter(text: str):
    if not text.startswith("---\n"):
        return None
    end = text.find("\n---", 4)
    if end == -1:
        return None
    return text[4:end]

def get_value(frontmatter: str, field: str):
    match = re.search(rf"^{re.escape(field)}\s*:\s*(.*)$", frontmatter, re.MULTILINE)
    if not match:
        return None
    return match.group(1).strip().strip('"').strip("'")

def main():
    errors = []

    for path in DOCS:
        text = path.read_text(encoding="utf-8")
        fm = parse_frontmatter(text)

        if fm is None:
            errors.append(f"{path}: missing frontmatter")
            continue

        for field in REQUIRED_FIELDS:
            if get_value(fm, field) is None:
                errors.append(f"{path}: missing field `{field}`")

        doc_type = get_value(fm, "doc_type")
        if doc_type and doc_type not in VALID_DOC_TYPES:
            errors.append(f"{path}: invalid doc_type `{doc_type}`")

        confidence = get_value(fm, "confidence")
        if confidence and confidence not in VALID_CONFIDENCE:
            errors.append(f"{path}: invalid confidence `{confidence}`")

        status = get_value(fm, "status")
        if status and status not in VALID_STATUS:
            errors.append(f"{path}: invalid status `{status}`")

        if status == "approved" and str(path).startswith("review/pending"):
            errors.append(f"{path}: pending document should not be approved")

    if errors:
        print("KB validation failed:")
        for err in errors:
            print(f"- {err}")
        sys.exit(1)

    print(f"KB validation passed. Checked {len(DOCS)} documents.")

if __name__ == "__main__":
    main()
EOF

chmod +x scripts/validate_kb.py
```

运行测试：

```bash
python scripts/validate_kb.py
```

提交初始版本：

```bash
git add .
git commit -m "Initialize OpenCode knowledge base workspace"
```

---

## 10. 配置模型

在服务器上进入知识库 repo：

```bash
cd /srv/company-kb
opencode
```

在 OpenCode TUI 中执行：

```text
/connect
```

按提示选择 provider，并填入 API key。

配置完成后执行：

```text
/models
```

选择默认模型。

退出后可以检查可用模型：

```bash
opencode models --refresh
opencode models
```

也可以用 `opencode run` 做一次非交互测试：

```bash
cd /srv/company-kb

opencode run "Read AGENTS.md and summarize your role in one paragraph. Do not edit files."
```

如果这个命令能正常输出，说明：

```text
OpenCode 可用；
模型配置可用；
当前用户的 provider 凭据可用；
当前 repo 的 AGENTS.md 可读。
```

这里要注意一个细节：后面 systemd 服务应该使用同一个 Linux 用户运行。不要用 `john` 执行 `/connect`，最后 systemd 却用 `root` 启动服务，否则可能读不到 provider 凭据。

---

## 11. 手动启动 OpenCode Web

先创建环境变量文件：

```bash
sudo mkdir -p /etc/company-kb
sudo nano /etc/company-kb/opencode-web.env
```

写入：

```bash
OPENCODE_SERVER_USERNAME=opencode
OPENCODE_SERVER_PASSWORD=change-this-to-a-strong-password
```

收紧权限(注意, 如果你的user 不是 root, 就不要写 root)：

```bash
sudo chown "$USER":"$USER" /etc/company-kb/opencode-web.env
chmod 600 /etc/company-kb/opencode-web.env
```

手动启动：

```bash
cd /srv/company-kb

set -a
source /etc/company-kb/opencode-web.env
set +a

opencode web --hostname 0.0.0.0 --port 4096
```

然后在浏览器打开：

```text
http://服务器IP:4096
```

登录后测试：

```text
请根据 AGENTS.md，创建一个 mp 模块的 sell-through 指标定义文档草稿，保存到 review/pending/sell-through.md。公式未知的地方写 Needs confirmation。
```

回到服务器检查：

```bash
cd /srv/company-kb
git diff
ls -la review/pending
python scripts/validate_kb.py
```

---

## 12. 配置防火墙

如果服务器使用 `ufw`，只允许局域网访问 4096。

假设局域网是 `192.168.0.0/16`：

```bash
sudo ufw allow from 192.168.0.0/16 to any port 4096 proto tcp
sudo ufw status
```

如果是 `10.0.0.0/8`：

```bash
sudo ufw allow from 10.0.0.0/8 to any port 4096 proto tcp
sudo ufw status
```

**不要**直接开放公网：

```bash
sudo ufw allow 4096/tcp
```

除非前面有 VPN、内网隔离、反向代理认证或公司 SSO。

---

## 13. 用 systemd 托管 OpenCode Web

确认 OpenCode 路径：

```bash
command -v opencode
```

生成服务文件：

```bash
OPENCODE_BIN="$(command -v opencode)"
CURRENT_USER="$(whoami)"

sudo tee /etc/systemd/system/opencode-kb-web.service > /dev/null <<EOF
[Unit]
Description=OpenCode Web for Company Knowledge Base
After=network.target

[Service]
Type=simple
User=$CURRENT_USER
WorkingDirectory=/srv/company-kb
EnvironmentFile=/etc/company-kb/opencode-web.env
ExecStart=/usr/bin/bash -lc 'exec "$OPENCODE_BIN" web --hostname 0.0.0.0 --port 4096'
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

启动：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now opencode-kb-web
sudo systemctl status opencode-kb-web
```

查看日志：

```bash
journalctl -u opencode-kb-web -f
```

检查端口：

```bash
ss -lntp | grep 4096
```

如果看到 `0.0.0.0:4096`，说明正在监听局域网。

---

## 14. 同事使用规则

给同事的说明可以保持很简单。

**访问**：

```text
http://服务器IP:4096
```

**用途**：

```text
这是知识库草稿整理工作台，不是正式知识库门户。
```

**请做**：

```text
1. 粘贴业务说明、截图说明或会议纪要；
2. 让 agent 按 AGENTS.md 整理成 Markdown；
3. 草稿统一保存到 review/pending/；
4. 不确定内容写入 Open Questions；
5. 不要让 agent 修改系统配置、删除文件、运行 sudo 命令。
```

**第一次使用**:

请先输入：

```text
/kb-start
```

这个命令会展示当前知识库工作台的用途、可创建的文档类型、草稿保存位置、安全边界和推荐提问方式。

如果不确定该怎么开始，也可以直接问：

```text
这个知识库工作台怎么用？
```

但更推荐使用 `/kb-start`，因为它会按照项目内置的 onboarding 规则进行引导。


**推荐提问**：

```text
请把下面内容整理成 MP 模块的 FAQ 文档，保存到 review/pending/xxx.md。
```

```text
请把这段指标说明整理成 metric_definition 文档，公式不确定处标记 Needs confirmation。
```

```text
请审查 review/pending/xxx.md 是否适合进入知识库。
```

**不要这样问**：

```text
帮我随便整理一下。
```

```text
你觉得公式是什么就补上。
```

```text
直接放到 approved。
```

```text
帮我删掉没用的文件。
```

---

## 15. 日常维护

负责人定期检查：

```bash
cd /srv/company-kb

git status
git diff
python scripts/validate_kb.py
```

确认草稿可以进入知识库后：

```bash
mv review/pending/sell-through.md knowledge/mp/metrics/sell-through.md
python scripts/validate_kb.py

git add .
git commit -m "Approve MP sell-through metric definition"
```

查看服务日志：

```bash
journalctl -u opencode-kb-web -n 200 --no-pager
```

重启服务：

```bash
sudo systemctl restart opencode-kb-web
```

---

## 16. 为什么暂时不装 MCP / LSP / 插件

第一版我不准备安装 MCP、LSP 或社区插件。

原因很简单：当前工作台只需要完成这些事：

```text
读写 Markdown；
整理 FAQ；
整理指标定义；
跑 validate_kb.py；
看 git diff；
进入人工 review。
```

内置工具、AGENTS.md、skills 和 permissions 已经足够。

MCP 适合后续接外部系统，比如 Dify dataset、内部只读 API、Jira、Confluence、向量库等。但第一版接 MCP 会显著增加权限和维护复杂度。

LSP 更适合代码项目。这个 repo 主要是 Markdown、少量 Python 脚本和 JSON schema，不需要一开始就接 LSP。

插件也一样，后续可以用来做 session logging、secret protection、自动通知、质量检查 hook。但第一版优先保证部署简单、权限清楚、流程可控。

---

## 17. 当前阶段的验收标准

第一版只要满足这些就算成功：

```text
[ ] OpenCode Web 能通过局域网访问
[ ] 访问需要用户名和密码
[ ] 模型配置可用
[ ] AGENTS.md 生效
[ ] 同事能生成 review/pending/*.md
[ ] git diff 能看到文档变更
[ ] validate_kb.py 能跑
[ ] agent 不会随意修改系统配置
[ ] agent 不会直接把草稿放到 approved
[ ] 负责人可以人工 review 和 commit
```

后续再考虑：

```text
approved 文档导入 Dify；
自动 chunking；
向量化；
Python Adapter 调用；
Java SaaS 页面展示；
复杂任务接 OpenCode runtime。
```

---

## 18. 总结

这个方案的核心是：

> 用 OpenCode Web 降低知识库冷启动成本；  
> 用 AGENTS.md / skills / commands / permissions 约束内容生产和工作流：  
> 用 Git / Review / Pipeline 保证知识资产可治理；  
> 用 Dify / Vector DB / Adapter 提供最终服务。

它是一种很轻的过渡方案。

如果后面知识库规模变大，再逐步补：

```text
审批流；
自动导入；
版本发布；
Dify 同步；
向量库；
评测集；
SaaS Copilot 正式入口。
```

但第一步，先让业务同事能低成本地产出结构化知识草稿。