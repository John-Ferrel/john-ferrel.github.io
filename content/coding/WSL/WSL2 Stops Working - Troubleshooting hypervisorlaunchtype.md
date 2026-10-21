---
title: "WSL2 Stops Working - Troubleshooting hypervisorlaunchtype"
draft: false
tags:
  - wsl
created: 2026-05-04 15:15
modified: 2026-10-22 00:05
aliases:
  - "coding/WSL/WSL2 突然不能用, 原因是 hypervisorlaunchtype"
---
原本正常使用的 WSL2 突然无法连接，VS Code 和 PowerShell 都无法启动 WSL。PowerShell 报错：

```text
当前计算机配置不支持 WSL2。
请启用“虚拟机平台”可选组件，并确保在 BIOS 中启用虚拟化。
错误代码: HCS_E_HYPERV_NOT_INSTALLED
```

检查发现，Windows 的 Hypervisor 启动项被关闭了。

管理员 PowerShell 里检查：

```powershell
bcdedit /enum | findstr -i hypervisorlaunchtype
```

结果看到：

```text
hypervisorlaunchtype    Off
```

WSL2 依赖 Windows 的虚拟化层，`hypervisorlaunchtype` 被设成 `Off` 之后，Windows 启动时不会加载 Hypervisor，WSL2 自然就起不来。

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
