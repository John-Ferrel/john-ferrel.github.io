---
title: OpenCode + GitHub - Agent for Code Review
tags:
  - opencode
  - github
  - agent
  - coding
draft: false
created: 2026-10-08 00:43
modified: 2026-10-08 22:39
---

目标是在 Pull Request 下通过 `/oc` 手动触发 OpenCode review，控制运行时机和检查范围。

```text
Pull Request
    ↓
/oc review
    ↓
GitHub Actions
    ↓
OpenCode
    ↓
PR comment
```

## 安装

初始化 GitHub integration：

```sh
opencode github install
```

安装向导会引导配置 GitHub App、workflow 和模型 API secrets。生成的 `.github/workflows/opencode.yml` 需要提交；`issue_comment` 要求 workflow 已存在于 default branch。也可以直接采用下文的 `GITHUB_TOKEN` 配置，无需安装 OpenCode GitHub App。参见 [OpenCode GitHub integration](https://opencode.ai/docs/github/)。

workflow 监听两类 comment：

```yaml
on:
  issue_comment:
    types: [created]

  pull_request_review_comment:
    types: [created]
```

PR 的 Conversation 页面普通评论使用 `issue_comment`；Files changed 中的行内评论使用 `pull_request_review_comment`，会附带文件路径、行号和 diff context。提交整个 review 对应的 `pull_request_review` 不在这个监听列表中。

例如，在 PR 下发布一条新评论：

```
/oc review this PR
```

这里只监听 comment 的 `created`，编辑旧评论不会重新触发。没有使用 `pull_request: synchronize` 自动触发，以避免每次 push 都运行一次 review。

## GitHub App token exchange

第一次配置后，workflow 可以启动，但 OpenCode 后续访问 GitHub 失败。

当时定位到 GitHub App token exchange 阶段。排查时应保留具体报错和 HTTP 状态码，区分 GitHub 认证与模型 API 认证；仅凭 workflow 已启动，无法判断是哪一层失败。

一种替代方式是直接使用 GitHub Actions 提供的 `GITHUB_TOKEN`：

```yaml
permissions:
  contents: write
  pull-requests: write
  issues: write

steps:
  - uses: anomalyco/opencode/github@latest
    env:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      # MODEL_API_KEY: ...
    with:
      use_github_token: true
      # model: ...
```

`use_github_token: true` 会跳过 OpenCode App 的 OIDC token exchange，也无需为此添加：

```yaml
permissions:
  id-token: write
```

上面是通用任务的权限示例。只负责 review 时可以使用 `contents: read`，限制通过这个 token 推送代码的能力；发布评论仍需相应的写权限。此模式下评论通常显示为 `github-actions[bot]`。配置依据见 [OpenCode 的 token 选项](https://opencode.ai/docs/github/#configuration)。

## 手动 review workflow

下面示例只接受 PR 下的评论，并在 job 层限制评论者身份。将其保存为目标 repository 的 `.github/workflows/opencode.yml`：

```yaml
name: OpenCode PR Review

on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]

jobs:
  review:
    if: |
      (github.event.issue.pull_request || github.event_name == 'pull_request_review_comment') &&
      contains(fromJSON('["OWNER", "MEMBER", "COLLABORATOR"]'), github.event.comment.author_association) &&
      (startsWith(github.event.comment.body, '/oc ') ||
       startsWith(github.event.comment.body, '/opencode '))
    runs-on: ubuntu-latest
    timeout-minutes: 20
    permissions:
      contents: read
      pull-requests: write
      issues: write
    steps:
      - name: Checkout repository
        uses: actions/checkout@v6
        with:
          fetch-depth: 1

      - name: Run OpenCode
        uses: anomalyco/opencode/github@latest
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        with:
          model: ${{ vars.OPENCODE_MODEL }}
          use_github_token: true
          share: false
```

使用前配置 repository variable `OPENCODE_MODEL`，值为可用的 `provider/model`。示例使用 Anthropic provider，需要配置 secret `ANTHROPIC_API_KEY`；使用其他 provider 时同步替换环境变量和 model。

这个 job 条件要求评论从 `/oc ` 或 `/opencode ` 开始，且后面有一个空格。裸 `/oc`、前置空白或不同大小写都不会通过这里的筛选；这比 [OpenCode 默认 mention 匹配](https://opencode.ai/docs/github/#configuration)更严格。使用上文的 `/oc review this PR` 即可。

这里保留 checkout 默认的 Git credentials，供 `GITHUB_TOKEN` 模式后续 fetch 使用。App token 模式的官方示例使用 `persist-credentials: false`，由 OpenCode 配置 Git 认证；切换模式时需要一起检查认证路径。参见 [checkout 的 credentials 说明](https://github.com/actions/checkout#usage) 与 [CLI 认证实现](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/cli/cmd/github.handler.ts)。

`author_association` 是 job 层的初步过滤，无法精确表达当前仓库的 write 权限。当前 CLI 还会检查触发者是否具有 `admin` 或 `write` 权限，部署时应确认所用版本的行为。`Do not modify the PR` 属于 prompt 约束；`contents: read` 限制远端写入，无法阻止 Agent 修改 runner 中的本地文件。

fork PR 也应单独验证。comment workflow 在 base repository 的权限上下文中运行，PR 文件、配置和指令都需要按不可信输入处理；不要在携带 secrets 的 job 中直接执行未经审查的 PR 脚本。初次部署优先用受控 PR 验证，再决定是否开放 fork PR。

示例保留官方使用的 `@latest` 便于说明。需要可复现运行时，应记录实际 CLI 版本，并评估固定 Action commit 和 CLI 版本；当前 Action 会自行获取最新 release，仅固定 Action commit 不等于固定 CLI。参见 [Action 定义](https://github.com/anomalyco/opencode/blob/dev/github/action.yml)。

## `issue_comment` 与 PR HEAD

`issue_comment` 的 `GITHUB_SHA` 和 `GITHUB_REF` 对应 default branch，初始 `actions/checkout` 通常也会落在这个 revision。因此不能直接把 `${{ github.sha }}` 当作 PR HEAD。参见 [GitHub 的 issue_comment 事件说明](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows#issue_comment)。

OpenCode 当前 CLI 会获取 PR 信息，再 fetch / checkout PR 分支；同仓库 PR 与 fork PR 分别处理。初始 checkout 到 default branch，不能单独证明 Agent review 了错误代码。相关实现见 [github.handler.ts](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/cli/cmd/github.handler.ts)。

```text
issue_comment
    ↓
初始 checkout default branch
    ↓
OpenCode 获取 PR 信息并切换分支
    ↓
Agent review → PR comment
```

上述实现按 branch ref 切换，严格锁定某个 SHA 的需求仍需额外处理。本文技术核对日期为 2026-10-08，源码链接指向可变的 `dev` 分支；实际运行以所安装 CLI 版本为准。

### 验证 checkout revision

让 Agent 在读取代码前输出以下信息；单独在 OpenCode step 前输出，只能检查初始 checkout：

```sh
git branch --show-current
git rev-parse HEAD
git log -1 --oneline
```

将 `git rev-parse HEAD` 与本次 review 记录的 PR head SHA 对比，报告中保留 reviewed SHA。运行期间若有新的 push，PR 页面显示的 HEAD 可能已经变化，需要区分本次检查的 commit 与最新 commit。

分支名只用于辅助定位；detached HEAD 或 fork PR 的本地分支名都不能代替 SHA 验证。

### 自定义 checkout 的处理

如果使用自定义 runner，或日志确认 Agent 没有切到正确 revision，需要先解析 PR，再获取 head SHA 并 checkout：

```
issue_comment
    ↓
PR number
    ↓
PR head SHA
    ↓
checkout
    ↓
OpenCode
```

`issue_comment` 的 PR number 来自 `github.event.issue.number`，并先检查 `github.event.issue.pull_request`；行内评论使用 `github.event.pull_request.number`。fork PR 的 head repository 与 base repository 可能不同，不能只用同仓库的 branch name 定位。

如果要求严格锁定 SHA，还需要确认后续工具不会再次切换到可移动的 branch ref。只在 OpenCode 前增加一次 checkout，无法保证运行期间 revision 保持不变。

## 真实 PR 验证

配置完成后，创建一个真实 PR，使用下文的完整 Review prompt 发布一条新 comment。

检查：

1. comment 是否正确触发 workflow；
2. Agent 开始 review 时的 SHA 是否等于本次记录的 PR head SHA；
3. OpenCode 是否读取到 PR diff 和相关 repository 文件；
4. review 是否发布到当前 PR，并包含 reviewed SHA；
5. 本次运行是否产生了代码修改，以及日志是否包含推送尝试。

其中第 2 项不能省略。

workflow 成功不代表 Agent review 的就是正确 revision。

## Review prompt

`/oc` 负责触发 Agent。只写简短的 review 请求，会把具体检查范围交给模型自行判断。

例如 `/oc review` 没有指定具体检查项，可以扩展为：

```
/oc review this PR.

Before reviewing, report the PR number, base SHA, and checked-out HEAD SHA.
Verify HEAD matches the PR head SHA obtained for this review.
If they differ, stop and explain the mismatch.
Read the PR diff and relevant surrounding code.

Focus on:
- correctness and logic errors
- regressions
- edge cases
- state / lifecycle issues
- concurrency and error handling
- security boundaries
- tests that do not actually prove the intended behavior

Only report evidence-based findings.
Point to the relevant code.
Do not modify the PR.
Do not edit files, commit, push, or run commands that change the repository.
If no concrete issues are found, say so and state any unverified areas.
```

如果目标是减少低价值建议，还可以进一步要求：

```
Do not report style preferences or optional refactors
unless they cause a concrete correctness or maintenance issue.
```

## 配置检查项

部署完成后至少确认：

```
[ ] /oc comment 能触发 workflow
[ ] GitHub authentication 正常
[ ] Agent 开始 review 时的 SHA == 本次记录的 PR head SHA
[ ] Agent 能读取 PR diff
[ ] Agent 能读取相关 repository context
[ ] review comment 返回当前 PR
[ ] review prompt 明确限制检查范围
[ ] 无代码修改或推送尝试
[ ] 评论者权限过滤符合预期
[ ] fork PR 与 secrets 的使用边界已确认
```

Actions 为绿色只说明 workflow 执行完成。验收时同时检查 reviewed SHA、diff/context 的读取情况和最终评论，才能确认本次 review 的对象与结果。
