---
title: WSL2 突然不能用, 原因是 hypervisorlaunchtype
draft: false
tags:
  - wsl
created: 2026-05-04 15:15
modified: 2026-07-12 00:00
---
今天遇到一个很迷惑的 WSL2 问题。

之前明明配置好、也一直能用 WSL，结果刚刚 VS Code 连不上 WSL。打开 PowerShell 直接报：

```text
当前计算机配置不支持 WSL2。
请启用“虚拟机平台”可选组件，并确保在 BIOS 中启用虚拟化。
错误代码: HCS_E_HYPERV_NOT_INSTALLED
```

不应该啊，我之前肯定成功配置的, 只是有一段时间没用。

最后查到是 Windows 的 Hypervisor 启动项被关了。

管理员 PowerShell 里检查：

```powershell
bcdedit /enum | findstr -i hypervisorlaunchtype
```

结果看到：

```text
hypervisorlaunchtype    Off
```

这就破案了。WSL2 依赖 Windows 的虚拟化层，`hypervisorlaunchtype` 被设成 `Off` 之后，Windows 启动时不会加载 Hypervisor，WSL2 自然就起不来。

修复命令：

```powershell
bcdedit /set hypervisorlaunchtype auto
```

然后重启电脑，WSL 恢复正常。

这个问题比较隐蔽：报错会让人先查 BIOS 虚拟化、WSL 安装和 Virtual Machine Platform，实际问题可能只是启动配置被改了。

可能原因包括：之前装过/调过 VirtualBox、VMware、安卓模拟器、Docker，或者某些教程为了兼容老虚拟机让你关过 Hypervisor，或者知识windows 更新了。

如果 WSL2 突然不能用了，而以前确认能用，先查：

```powershell
bcdedit /enum | findstr -i hypervisorlaunchtype
```

如果是 `Off`，改回 `auto`，重启，可能就好了。
