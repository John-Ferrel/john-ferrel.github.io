---
title: Streaming Windows to iPad - Sunshine + Moonlight
draft: false
tags:
  - gaming
  - sunshine
  - moonlight
  - ipad
created: 2026-10-07 21:19
modified: 2026-10-08 22:39
---

> 想躺在床上用 iPad 玩 Windows 上的游戏，而且不限 Steam。

主要是 RimWorld、模拟经营，以及一些只支持键盘鼠标的简单 3D 游戏，不需要公网串流，也不想折腾复杂的远程桌面方案。

**Windows → Sunshine → 局域网 → Moonlight → iPad**

Steam Link 可以串流，但 Sunshine + Moonlight 更像是把整个 Windows 桌面送到 iPad，不需要先把每个程序加进 Steam。

Sunshine 是 Windows 端 Host，Moonlight 是 iPad 端 Client。

---

## 1. 安装 Sunshine

Windows 安装 Sunshine 后，默认管理页面：

```
https://localhost:47990
```

首次访问本机这个地址时，浏览器可能提示自签名证书警告；确认访问的是自己安装的 Sunshine 管理页面后继续。

我的基础配置基本保持默认：

```
Encoder: Auto
Capture: Auto
HEVC: Auto
HDR: Off

Mouse: Enabled
Keyboard: Enabled

```


---

## 2. iPad 安装 Moonlight

Windows 和 iPad 接入同一个局域网。

Moonlight 正常情况下会自动发现 Sunshine。

如果没有，也可以手动输入 Windows 的局域网 IP，例如：

```
192.168.xx.xxx
```

只填 IP，不需要协议和端口。

---

## 3. iPad 本地网络权限

这是我整个过程中漏掉的设置。

现象：

- iPad Safari 能访问 Sunshine
- Moonlight 搜不到电脑
- 手动输入 IP 也连接失败

解决方案：

iPad 打开：

**设置 → 隐私与安全性 → 本地网络 → Moonlight**

确保 Moonlight 的本地网络权限已经开启。

---

## 4. 配对 Sunshine 和 Moonlight

Moonlight 找到电脑后，点击主机，会显示一个 PIN。

Windows 打开：

```
https://localhost:47990
```

进入 Sunshine 的 PIN 页面，输入这个 PIN。

完成以后，Moonlight 就可以直接启动 Desktop。

---

## 5. Moonlight 画质设置

我的 PC 显示器是：

```
2560 × 1440
```

客户端是 2021 iPad Pro。

最后使用：

```
Resolution: 2560 × 1440
FPS: 60
Bitrate: 30–40 Mbps
Codec: HEVC
HDR: Off
Stretch Video: Off
```

对我玩的这些普通游戏已经足够。这组参数可以作为起点，再根据局域网延迟、丢帧和画面质量调整码率。

PC 显示器通常是 16:9，而 iPad 更接近 4:3，所以串流时出现黑边很正常。不要为了铺满屏幕开启 Stretch，否则画面会被强行拉伸。

后续如果非常在意黑边，再考虑自定义分辨率或虚拟显示器。

---

## 6. 鼠标游戏：Touchscreen 还是 Trackpad

Moonlight 有两种触摸模式。

### Trackpad Mode

把整个 iPad 当成笔记本触控板。

```
滑动        → 移动鼠标
点击        → 左键
长按拖动    → 左键拖动
双指操作    → 滚动 / 右键
```

适合：

- Windows 桌面
- 小按钮
- 精确定位

### Touchscreen Mode

手指位置直接对应 Windows 屏幕坐标。

```
点哪里      → 左键点哪里
长按        → 右键
按住拖动    → 鼠标拖动
```

更像真正的触屏游戏。

实际用下来，两种都值得保留：

**粗操作用 Touchscreen，精确操作用 Trackpad。**

Trackpad 的右键具体操作是按住一根手指，再用第二根手指轻点；双指竖向拖动用于滚动。Touchscreen 的长按用于右键，拖动使用按下后移动手指。手势可对照 [Moonlight Setup Guide](https://github.com/moonlight-stream/moonlight-docs/wiki/Setup-Guide#touchscreen-controls)。

---

## 7. 如果游戏只支持 WASD 怎么办

有些简单 3D 游戏只支持：

```
WASD
Shift
Space
E
Mouse
```

Moonlight 的屏幕键盘显然不适合持续按 WASD。

解决方法是：

```
iPad 虚拟手柄
        ↓
Moonlight
        ↓
Sunshine
        ↓
Windows 虚拟 Xbox 手柄
        ↓
AntiMicroX
        ↓
WASD + Mouse
```

简单说，就是先让 iPad 模拟一个手柄，再把这个手柄翻译成键盘鼠标。

---

## 8. Sunshine 开启虚拟手柄

Sunshine：

**Configuration → Input**

设置：

```
Controller: Enabled
Gamepad Driver: ViGEmBus
Gamepad: Xbox 360
```

本文记录的是 ViGEmBus + Xbox 360 路径。对应版本如果还没有 ViGEmBus，需要先安装（控制台会提示）。Sunshine 不同版本的后端和界面名称可能变化；新版文档已列出 Virtual HID Driver 等选项，安装时按所用版本确认，参见 [Sunshine Input 配置](https://docs.lizardbyte.dev/projects/sunshine/master/md_docs_2configuration.html#input)。

完成后可能需要重启 Windows。

---

## 9. Moonlight 打开屏幕手柄

Moonlight：

```
On-Screen Controls: Full
```

重新串流后，屏幕上会出现：

```
左摇杆   右摇杆
方向键   ABXY
LT / RT
LB / RB
Start / Back
```

---

## 10. 确认 Windows 收到了手柄

Windows：

```
Win + R
```

输入：

```
joy.cpl
```

保持 Moonlight 正在串流，然后在 iPad 上拨动虚拟摇杆。

如果一切正常，会看到：

```
Xbox 360 Controller for Windows
```

说明已经打通控制器链路：

```
iPad
↓
Moonlight
↓
Sunshine
↓
ViGEmBus
↓
Windows
```

---

## 11. 用 AntiMicroX 把手柄映射成键鼠

AntiMicroX 的作用就是：

> Gamepad → Keyboard / Mouse

我先做了一套很通用的 3D 游戏映射：

```
左摇杆 ↑    → W
左摇杆 ↓    → S
左摇杆 ←    → A
左摇杆 →    → D

右摇杆      → Mouse movement

A           → Space
X           → E
Y           → F
B           → C

LB          → Shift
RB          → Ctrl

LT          → Right Mouse Button
RT          → Left Mouse Button

Back        → Tab
Start       → Esc
```

最终操作逻辑就是：

```
左拇指：移动
右拇指：转镜头

RT：左键
LT：右键

A：跳跃
X：互动
LB：Shift
```

这样一个完全没有 Gamepad 支持的键鼠游戏，也可以变成接近手机 3D 游戏的操作方式。

不同游戏再各自保存一个 AntiMicroX Profile 即可。

---

## 常见问题

### Moonlight 完全发现不了电脑

先检查：

```
iPad
设置
→ 隐私与安全性
→ 本地网络
→ Moonlight = On
```

Safari 能访问管理页面时，也要检查 Moonlight 自己的本地网络权限。管理页面可访问只说明该地址的 HTTPS 连通，不能证明串流所需连接都正常。

### Sunshine 报 CSRF Protection Error

如果自己改过 `bind_address`，先检查该改动，并用下面的本机地址重新访问。`bind_address` 用于选择监听地址；遇到 CSRF 报错，还应核对浏览器实际访问的地址以及反向代理配置，仅凭报错无法确定原因。

管理 Sunshine 尽量统一使用：

```
https://localhost:47990
```

### iPad 已经显示虚拟手柄，但 Windows 看不到

检查：

```
Controller = Enabled
Gamepad Backend = ViGEmBus
```

然后用：

```
joy.cpl
```

验证。

### 游戏根本不支持手柄


```
Moonlight Gamepad
→ Sunshine
→ ViGEmBus
→ AntiMicroX
→ Keyboard / Mouse
```

本质上游戏最后收到的仍然只是普通键盘鼠标输入。

---
