---
title: "Ubuntu 24.04 上运行 cc-connect + Codex：PATH、Sandbox 与 Git"
tags:
  - cc-connect
  - codex
  - ubuntu
  - sandbox
  - git
draft: false
created: 2026-08-30 00:00
modified: 2026-10-08 23:23
---

这篇整理 cc-connect + Codex 的分层排障步骤。同一次配置过程中的错误推断、实际日志与 Git 权限问题，记录在 [[CC-Connect + Codex - Error and Mistakes]]。

环境：

```
Ubuntu 24.04
cc-connect v1.4.1
Codex CLI 0.151.0

Telegram
  ↓
cc-connect
  ↓
Codex CLI
  ↓
Git repository
```

交互式 shell 中直接运行 Codex 正常，但接入 cc-connect daemon 后，先后遇到了三个问题：

1. daemon 找不到 `codex`
2. Codex sandbox 无法启动
3. `network_access = true` 配置后，测试结果仍然显示无法联网

这几个问题来自不同层。

## 1. daemon 找不到 Codex

Codex 安装在：

```
~/.local/bin/codex
```

SSH 登录后可以直接执行：

```
which codex
codex --version
```

但 cc-connect daemon 中无法调用。

原因是 systemd user service 的 `PATH` 和交互式 shell 不一致。

查看 service 环境：

```
systemctl --user show cc-connect -p Environment
```

给 cc-connect 增加 PATH：

```
systemctl --user edit cc-connect
```

加入：

```
[Service]
Environment="PATH=/home/john/.local/bin:/usr/local/bin:/usr/bin:/bin"
```

重新加载：

```
systemctl --user daemon-reload
systemctl --user restart cc-connect
```

检查：

```
cc-connect daemon status
cc-connect daemon logs -n 50
systemctl --user show cc-connect -p Environment
```

示例中的 `/home/john` 需要替换为实际运行 service 的用户目录。`Environment=` 中使用绝对路径，不依赖 shell 展开 `~` 或 `$PATH`。

这里需要验证 daemon 实际拿到的环境。当前 SSH shell 中的值可以通过下面的命令查看：

```
echo $PATH
```

后者不能证明 systemd service 能找到同一个 executable。

## 2. Codex sandbox 在 Ubuntu 24.04 上启动失败

PATH 修好后，Codex 被正常调用，但 sandbox 初始化失败。

先检查 user namespace 相关参数：

```
sysctl kernel.unprivileged_userns_clone
sysctl user.max_user_namespaces
sysctl kernel.apparmor_restrict_unprivileged_userns
```

这次排查定位到以下限制；仅看到值为 `1` 还不足以确认故障原因，需要结合 sandbox 报错和验证结果：

```
kernel.apparmor_restrict_unprivileged_userns = 1
```

Ubuntu 24.04 的 AppArmor 对 unprivileged user namespace 有额外限制。

先临时修改进行验证：

```
sudo sysctl -w kernel.apparmor_restrict_unprivileged_userns=0
```

然后重新运行 Codex。这条命令会关闭整机范围的 AppArmor unprivileged user namespace 限制，影响不限于 Codex。验证后恢复修改前的值；没有写入持久化配置时，重启后按系统配置恢复。

对于使用 `bwrap` 的 Codex 版本，长期处理应优先检查其 AppArmor profile。[官方 sandbox 文档](https://learn.chatgpt.com/docs/sandboxing)提供了 Ubuntu 24.04 加载 `bwrap-userns-restrict` profile 的方法。

如果修改后 sandbox 可以正常启动，就已经定位到 AppArmor / user namespace 这一层，不需要先重装 Codex、cc-connect 或 reboot。

这里最重要的是先区分：

```
cc-connect 无法启动 Codex
```

和：

```
Codex 已启动，但 Linux sandbox 初始化失败
```

两者日志可能连续出现，但不是同一个问题。

## 3. `network_access = true` 为什么仍然无法联网

Codex 配置中已经设置：

```
[sandbox_workspace_write]
network_access = true
```

但测试时仍然发现 sandbox 内没有网络。

最开始使用 sandbox helper 测试，入口是：

```
codex sandbox
```

这里的测试本身有问题。

[[CC-Connect + Codex - Error and Mistakes#`workspace-write` 网络：测试条件错了]] 保存了当时的完整对照记录：

```bash
codex sandbox -- curl -I https://example.com
codex -c 'sandbox_mode="workspace-write"' sandbox -- curl -I https://example.com
```

前一次返回 DNS 解析失败，后一次返回 `HTTP/2 200`。这些命令记录的是本文环境中的历史验证；复现时先查看已安装版本的 help，核对平台子命令和参数，不据此推断所有版本的默认 mode。

`[sandbox_workspace_write]` 只配置：

```
workspace-write
```

sandbox mode。

如果测试命令实际启动的是另一个 mode，那么：

```
network_access = true
```

不会应用到正在测试的 sandbox。

因此需要明确使用对应的 sandbox mode，再验证网络。

也就是说，应该检查的是：

```
实际 sandbox mode
        ↓
对应 config section
        ↓
network_access
```

仅看到配置文件里存在下面这项，仍不足以判断当前运行的策略：

```
network_access = true
```

还要核对实际加载的配置文件、profile 和命令行覆盖项。

修正测试方式后，`workspace-write` 下的网络访问正常。

## 4. `workspace-write` 不等于完整 Git 写权限

网络恢复后，还有一个容易混在一起的问题。

Codex 使用 `workspace-write` 时，可以修改 repository 中的普通工作区文件：

```
repo/
├── src/       writable
├── tests/     writable
├── README.md  writable
└── .git/      protected
```

`workspace-write` 会保护可写根目录中的 `.git` 等特殊路径，普通工作区可写不会自动授予 Git metadata 写权限。具体行为以实际版本和生效的权限策略为准，参见[官方 sandbox 说明](https://learn.chatgpt.com/docs/sandboxing)。

因此可能出现：

```
git diff
git status
```

正常，但涉及 Git metadata 写入的操作失败。

例如某些：

```
commit
branch
checkout
index / refs updates
```

会触及 `.git`。

这和：

```
Codex 能不能修改代码
```

是两个不同的权限边界。

如果 Agent 的职责只是：

```
读取 repository
修改文件
运行测试
返回 diff
```

`workspace-write` 已经覆盖主要需求。

如果要求 Agent 自己执行完整 Git workflow，则还需要单独处理 `.git` 写入权限，而不能根据“源码文件可以写”推断 Git 操作也可以写。

## 5. 分层验证

整个链路可以拆开测试。

### Codex executable

```
which codex
codex --version
```

### systemd PATH

```
systemctl --user show cc-connect -p Environment
```

### cc-connect

```
cc-connect daemon status
cc-connect daemon logs -n 50
```

### Linux sandbox

```
sysctl kernel.unprivileged_userns_clone
sysctl user.max_user_namespaces
sysctl kernel.apparmor_restrict_unprivileged_userns
```

### sandbox mode

确认实际运行的 mode 与配置段一致：

```
[sandbox_workspace_write]
network_access = true
```

### Git

分别测试：

```
git status
```

普通文件写入，以及需要修改 `.git` 的操作。

不要把它们合成一个“Codex 权限是否正常”的测试。

最终需要确认的是这几层：

```
systemd
  └── PATH 能找到 codex

Linux / AppArmor
  └── Codex 能创建 sandbox

Codex sandbox mode
  ├── workspace 可写
  └── network_access 生效

Git
  ├── worktree 操作
  └── .git metadata 操作
```

出现错误时，从失败所在的这一层继续查，比重新调整整个 cc-connect / Codex 配置更容易定位问题。
