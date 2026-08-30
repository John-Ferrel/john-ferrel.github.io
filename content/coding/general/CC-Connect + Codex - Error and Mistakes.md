---
title: CC-Connect + Codex - Error and Mistakes
draft: false
tags:
  - ai
  - agent
  - git
  - codex
  - cc-connect
  - linux
created: 2026-08-30 20:15
modified: 2026-08-30 20:15
---
最近因为 DeepSeek 涨价，我把一台远程服务器上的 Telegram coding bot 从 OpenCode 切到了 Codex。( 直接使用 openai api 在 opencode 上缓存命中似乎有问题)

```text
Telegram → cc-connect → Codex CLI → Git repo
```

环境是 Ubuntu 24.04、cc-connect v1.4.1、Codex CLI 0.151.0。这个 bot 会长期在线，需要能够修改项目、执行测试、访问网络，并使用 Git 管理开发过程。

切换过程中遇到了几个独立问题。PATH 和 Ubuntu sandbox 很快定位；后面的 network debug 因测试条件错误被带偏，GPT-5.6 Sol 又基于这个错误结果连续给出了配置、版本和权限层面的推断，其中还包括一次无意义的整机重启。

## daemon 找不到 Codex

Codex 通过官方脚本安装在 `~/.local/bin`。SSH shell 可以正常找到，但 cc-connect 启动时报错：

```bash
$ which codex
/home/john/.local/bin/codex

cc-connect:
codex: "codex" CLI not found in PATH
```

**交互式 shell 的 PATH 和 systemd user service 的 PATH 不同。**

给 cc-connect service 补上 `~/.local/bin` 后，Codex 可以正常启动。

期间为了排除安装方式的问题，又通过 npm 安装过一份 Codex。确认原因与安装方式无关后，最终只保留：

```bash
/home/john/.local/bin/codex
```

避免 shell 和 daemon 命中不同版本。

## Ubuntu 24.04 阻止 Codex sandbox

Codex 的 Linux sandbox 依赖 unprivileged user namespace。这台服务器第一次创建 namespace 时失败：

```bash
$ unshare --user --map-root-user --net -- ...
unshare: write failed /proc/self/uid_map: Operation not permitted

$ sysctl kernel.apparmor_restrict_unprivileged_userns
kernel.apparmor_restrict_unprivileged_userns = 1
```

Ubuntu 24.04 的 AppArmor 默认限制了 unprivileged user namespace。

将：

```bash
kernel.apparmor_restrict_unprivileged_userns=0
```

应用后，`unshare` 和 `codex sandbox` 都恢复正常。随后把配置写入 `/etc/sysctl.d/` 持久化。

这个设置是系统级的，会影响普通用户进程创建 user namespace，不只是 Codex。对于共享服务器或不可信 workload，需要单独评估这一安全边界。

这里还发生了一次明确的错误操作。

`sysctl -w` 已经即时生效，`unshare` 和 Codex sandbox 也已经验证通过。此时 **GPT-5.6 Sol 仍建议执行 `sudo reboot`**，我按建议重启了整台服务器。

这次 reboot 没有参与修复，也没有提供新的诊断信息，却让服务器上的其他服务一起经历了重启。这里需要的操作到 sysctl 持久化为止；如需重新加载 cc-connect，也只需要处理对应 service。

## `workspace-write` 网络：测试条件错了

Codex 配置里已经开启：

```toml
[sandbox_workspace_write]
network_access = true
```

第一次测试得到：

```bash
$ codex sandbox -- curl -I https://example.com
curl: (6) Could not resolve host: example.com
```

GPT-5.6 Sol 随后把 debug 带到了 Codex 0.151.0 的配置行为、CLI override、permission profile、wrapper，以及 cc-connect mode。

这些方向都建立在同一个假设上：`network_access = true` 没有生效。

真正的问题在测试条件。

这个配置属于 `workspace-write`，而前面的命令运行的是默认 sandbox。改成对应的 mode：

```bash
$ codex -c 'sandbox_mode="workspace-write"' sandbox -- curl -I https://example.com
HTTP/2 200
```

网络一直是正常的。

**测试的是默认 sandbox，配置针对的却是 `workspace-write`。**

前一个结果不能用来判断后一个配置是否生效。这个错误对照随后又被继续解释成版本或权限问题，才把 debug 范围不断扩大。

对于有多种 sandbox / permission mode 的 agent runtime，测试环境必须和实际运行环境一致。

## `workspace-write` 与 Git

网络确认后，Codex 已经可以修改项目、运行测试和访问网络。

随后在提交修改时失败：

```bash
git add:
Unable to create '.git/index.lock': Read-only file system
```

`workspace-write` 的边界大致是：

```text
repo/
├── source    RW
├── tests     RW
├── docs      RW
└── .git      RO
```

工作区可以写，Git metadata 受到保护。

因此 `git status`、`git diff` 可以正常使用；`git add`、`commit`、`branch`、`stash`、`reset`、`rebase` 等操作需要修改 `.git`，会受到限制。

对于交互式 coding，这个边界成立：

**Codex 修改 workspace，用户维护 Git 状态。**

长期运行的 coding bot 则不同。它需要自己维护完整的开发状态：

```text
branch → edit → test → commit → continue / rollback
```

Git 在这里同时承担版本管理、checkpoint 和回滚。如果 agent 可以改完整个 workspace，却不能维护 Git 状态，开发循环会停在版本管理这一层。

我随后通过 cc-connect 切换了 `yolo`，当前 session 和新 session 中仍然观察到 `.git` 只读。由于还没有完成独立 Codex CLI bypass 的对照测试，这部分暂时没有继续判断具体原因。

## Sandbox 应该包住什么

到这里，问题已经不只是某个 Git 命令能不能执行。

如果 Codex sandbox 作为主要隔离层，它需要同时容纳：

```text
filesystem
network
Git state
```

另一种部署方式是把隔离边界往外移：

```text
host
└── isolated user / container
    ├── Codex
    └── full Git repo
```

这样完整 repository，包括 `.git`，都属于 agent 的开发环境；其他项目、凭据和宿主机资源由 Linux user、container 或 VM 隔离。

这两种方式对应不同的运行模型：一种更适合 human-in-the-loop，另一种更接近长期自主工作的 coding agent。

## Debug 过程里暴露的问题

这次 debug 中，几个偏差具有相同的结构：

```text
观察到现象
→ 提出假设
→ 假设尚未验证
→ 开始修改配置
→ 根据新的系统状态继续推理
```

network 是最完整的例子。

如果先写清楚：

```text
事实：默认 sandbox 无网络
配置：network_access 属于 workspace-write
待验证：workspace-write 是否有网络
```

下一步只需要一个对照实验。

整机 reboot 则属于另一类错误：**操作范围超过了证据范围。** 内核参数已经即时生效并完成验证，没有理由继续扩大到整台服务器。

在远程服务器上用 AI debug，我现在会固定三个状态：

* 已确认事实
* 当前假设
* 用于区分假设的实验

实验完成之前，不把假设直接转换成配置修改。

操作范围同样按层级递增：

```text
进程参数 → 应用配置 → 单个服务 → 系统配置 → 整机
```

当前层能够完成验证和修复，就不进入下一层。

这次已经确认了 cc-connect daemon PATH、Ubuntu user namespace、Codex sandbox 和 `workspace-write` 网络的实际行为。

剩下的核心问题在 Git：**一个需要自主 branch、commit 和 rollback 的 coding agent，完整版本库应该放在它的哪一层安全边界里。**
