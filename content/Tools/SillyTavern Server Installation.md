---
title: SillyTavern Server Installation
draft: false
tags:
  - llm
  - server
created: 2026-04-14 21:52
modified: 2026-07-12 00:00
---

**环境:Ubuntu 24.04 + WireGuard + Docker + SillyTavern**

## 1. 目标

在一台 Ubuntu 24.04 服务器上部署:

- WireGuard VPN
- Docker
- SillyTavern

使用方式:

- 用户先连接 WireGuard
- 再通过 VPN 内网地址访问 SillyTavern
- 不直接把 SillyTavern 暴露到公网

SillyTavern 官方支持 multi-user, 但也明确提醒其用户隔离不是强安全边界, 且不建议直接将实例暴露到互联网. 

SillyTavern 官方文档: [SillyTavern Documentation](https://docs.sillytavern.app/administration/multi-user/)

---

## 2. 服务器信息

示例配置:

- 服务器公网 IP: `xxx.xxx.xxx.xxx`
- WireGuard 监听端口: `51820/udp`
- WireGuard 网段: `10.xx.xx.0/24`
- 服务器 VPN 地址: `10.xx.xx.1/24`
- 客户端 1 VPN 地址: `10.xx.xx.2/24`
- SillyTavern 端口: `8000`

---

## 3. 系统初始化

更新系统:

```bash
sudo apt update
sudo apt full-upgrade -y
sudo reboot
```

安装基础工具:

```bash
sudo apt install -y curl wget git nano ufw ca-certificates gnupg lsb-release
```

---

## 4. 安装 Docker

### 4.1 推荐:使用 Docker 官方源

移除旧包:

```bash
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do
  sudo apt remove -y $pkg
done
```

添加 Docker 官方仓库:

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL --retry 3 https://download.docker.com/linux/ubuntu/gpg -o /tmp/docker.asc
gpg --show-keys /tmp/docker.asc
sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg /tmp/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

安装 Docker:

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

设置开机启动:

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

检查版本:

```bash
docker --version
docker compose version
```

```bash
root@xxx:~# docker --version
Docker version 29.4.0,  build 9d7ad9f

root@xxx:~# docker compose version
Docker Compose version v5.1.2
```

---

## 5. 安装 WireGuard

安装:

```bash
sudo apt install -y wireguard
```

生成服务器密钥:

```bash
umask 077
wg genkey | tee server_private.key | wg pubkey > server_public.key
```

查看服务器公钥和私钥:

```bash
cat server_public.key
cat server_private.key
```

---

## 6. 配置 WireGuard 服务器

编辑配置文件:

```bash
sudo nano /etc/wireguard/wg0.conf
```

写入:

*此时先完成接口配置即可，生成客户端公钥后再补充 `[Peer]`段*

```ini
[Interface]
Address = 10.xx.xx.1/24
ListenPort = 51820
PrivateKey = <服务器私钥>
SaveConfig = false

```

建议使用一个未被本地网络占用的私有网段，例如 `10.66.57.0/24`

---

## 7. 开启 IP 转发

编辑:

```bash
sudo nano /etc/sysctl.conf
```

确保存在:

```ini
net.ipv4.ip_forward=1
```

生效:

```bash
sudo sysctl -p
```

---

## 8. 配置防火墙

只开放 SSH 和 WireGuard:

```bash
sudo ufw allow OpenSSH
sudo ufw allow 51820/udp
sudo ufw enable
sudo ufw status
```

**不要开放 8000 到公网.**

---

## 9. 启动 WireGuard

```bash
sudo systemctl enable wg-quick@wg0
sudo systemctl restart wg-quick@wg0
```

检查状态:

```bash
sudo wg
ip addr show wg0
```

成功时应看到:

- `interface:wg0`
- `listening port:51820`
- `inet 10.xx.xx.1/24`

---

## 10. 生成客户端配置

为客户端生成密钥:

*私钥只用于填配置，不要发送给其他人，不要截图，不要放进公开文档。*

```bash
wg genkey | tee client1_private.key | wg pubkey > client1_public.key

cat client1_private.key
cat client1_public.key
```

把客户端公钥填回服务器 `/etc/wireguard/wg0.conf` 的 `[Peer]` 段,  

```ini
[Interface]
Address = 10.xx.xx.1/24
ListenPort = 51820
PrivateKey = <服务器私钥>
SaveConfig = false

[Peer]  
PublicKey = <客户端1公钥>
AllowedIPs = 10.xx.xx.2/32
```

重启:

```bash
sudo systemctl restart wg-quick@wg0
```

客户端配置示例:

```ini
[Interface]
PrivateKey = <客户端1私钥>
Address = 10.xx.xx.2/24
DNS = 8.8.8.8

[Peer]
PublicKey = <服务器公钥>
Endpoint = xx.xx.xx.xx:51820
AllowedIPs = 10.xx.xx.0/24
PersistentKeepalive = 25
```

导入到 Windows / macOS / Linux / Android / iPhone 的 WireGuard 客户端. 

连接成功后, 客户端应能访问:

```text
PS C:\Users\xxx> ping 10.xx.xx.1

正在 Ping 10.xx.xx.1 具有 32 字节的数据:
来自 10.xx.xx.1 的回复:字节=32 时间=19ms TTL=64
来自 10.xx.xx.1 的回复:字节=32 时间=20ms TTL=64
来自 10.xx.xx.1 的回复:字节=32 时间=20ms TTL=64
来自 10.xx.xx.1 的回复:字节=32 时间=20ms TTL=64
```

---

## 11. 安装 SillyTavern

创建目录:

```bash
mkdir -p ~/sillytavern
cd ~/sillytavern
```

创建 `docker-compose.yml`:

```bash
nano docker-compose.yml
```

写入:

```yaml
services:
  sillytavern:
    image: ghcr.io/sillytavern/sillytavern:latest
    container_name: sillytavern
    restart: unless-stopped
    ports:
      - "10.xx.xx.1:8000:8000"
    volumes:
      - ./config:/home/node/app/config
      - ./data:/home/node/app/data
      - ./plugins:/home/node/app/plugins
```

启动:

```bash
docker compose up -d
docker compose logs -f
```

这样 SillyTavern 只绑定在 VPN 地址 `10.xx.xx.1` 上, 不直接绑定公网地址. 

---

## 12. 配置 SillyTavern

首次启动后, 配置文件生成在:

```bash
~/sillytavern/config/config.yaml
```

编辑:

```bash
nano ~/sillytavern/config/config.yaml
```

建议最小配置:

```yaml
enableUserAccounts: true
enableDiscreetLogin: true
listen: true
port: 8000
whitelist:
  - 10.xx.xx.0/24
```

重启容器:

```bash
cd ~/sillytavern
docker compose restart
```

说明:

- `enableUserAccounts: true`: 开启多用户
- `enableDiscreetLogin: true`: 隐藏登录页用户列表
- `listen: true`: 允许外部访问容器服务
- `whitelist`: 白名单,  开放对应网络段

SillyTavern 官方文档确认 multi-user 可让不同用户拥有各自设置和数据. 

---

## 13. 访问 SillyTavern

客户端先连接 WireGuard, 再访问:

```text
http://10.xx.xx.1:8000
```

使用 VPN 内网地址，不是公网 IP 或端口。

首次登录后:

1. 创建管理员账号
2. 创建普通用户账号
3. 每个用户单独使用自己的账号

---

## 14. 配置 hosted API

在 SillyTavern 页面中:

1. 打开 API Connections
2. 选择相应的 API 类型
3. 填写 base URL 和 API key
4. 测试连接

如果 OpenAI-compatible 网关部署在宿主机，而 SillyTavern 跑在 Docker 里，容器应通过 `host.docker.internal` 访问宿主机服务。

---

## 15. 新增一个用户的流程

每新增一位朋友, 需要做两件事:

### 15.1 增加一个 WireGuard peer

可以参看 [[Wireguard 简易使用#快速新增一个客户端]]

- 生成新的客户端密钥
- 分配新的 VPN 地址, 例如 `10.xx.xx.3/32`
- 在服务器 `/etc/wireguard/wg0.conf` 中增加一个 `[Peer]`

### 15.2 在 SillyTavern 中创建一个新账号

- 单独用户名
- 单独密码
- 不共用账号

Ubuntu 官方 WireGuard 文档也把“新增 peer”作为常规操作. ([Ubuntu](https://ubuntu.com/server/docs/how-to/wireguard-vpn/common-tasks/))

---

## 16. 常用运维命令

查看 WireGuard:

```bash
sudo wg
watch wg
```

查看 SillyTavern:

```bash
cd ~/sillytavern
docker compose ps
docker compose logs -f
```

更新 SillyTavern:

```bash
cd ~/sillytavern
docker compose pull
docker compose up -d
```

备份:

```bash
tar czf sillytavern-backup-$(date +%F).tar.gz ~/sillytavern
```

---

## 17. 故障排查

### 17.1 Docker 官方源安装失败

常见原因:

- `download.docker.com` 网络异常
- GPG key 下载失败
- 报 `NO_PUBKEY`

解决方式:

- 重新下载 GPG key
- 或临时使用 Ubuntu 自带 `docker.io`

### 17.2 WireGuard 显示连接但无法 ping `10.xx.xx.1`

重点检查:

```bash
sudo wg
cat /etc/wireguard/wg0.conf
```

若服务器端没有 `[Peer]`, 客户端就不会真正被接受. 

### 17.3 SillyTavern 打不开

检查:

```bash
cd ~/sillytavern
docker compose ps
docker compose logs -f
ss -tulpn | grep 8000
```

### 17.4 VPN 通了但 ST 连不上

确认 `docker-compose.yml` 绑定的是:

```yaml
ports:
  - "10.xx.xx.1:8000:8000"
```

并且客户端访问的是:

```text
http://10.xx.xx.1:8000
```

---

## 18. 安全建议

- 只给熟人使用
- 不把 SillyTavern 直接暴露到公网
- 不开放 8000 公网端口
- 不共享管理员账号
- 不安装来源不明的扩展和 server plugins
- 定期备份 `~/sillytavern`

官方文档明确提示 third-party extensions 和 server plugins 有安全风险, 且 server plugins 不做 sandbox. 

---

## 19. 最小闭环

成功部署的最小标准:

1. 服务器 `sudo wg` 正常
2. 客户端连接 WireGuard 成功
3. 客户端能访问 `10.xx.xx.1`
4. 浏览器能打开 `http://10.xx.xx.1:8000`
5. 能登录 SillyTavern 并连接 hosted API




---

### Logs

**docker 安装失败**

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

日志里显示:

- `curl:(35) Recv failure:Connection reset by peer`
- `gpg:no valid OpenPGP data found.`
- 后面 `apt update` 又报 `NO_PUBKEY 7EA0A9C3F273FCD8`

Docker 的 GPG key **没有下载成功**，`/etc/apt/keyrings/docker.gpg` 因此为空或无效。
Docker 源虽然已写入，但系统缺少对应公钥，仓库签名校验会失败。

**修复**

按下面这组命令重新来, 不要沿用刚才那个坏文件. 

1. 删掉坏的 key 文件

```bash
sudo rm -f /etc/apt/keyrings/docker.gpg
```

2. 重新下载 key

```bash
sudo mkdir -p /etc/apt/keyrings  
curl -fsSL --retry 3 https://download.docker.com/linux/ubuntu/gpg -o /tmp/docker.asc  
gpg --show-keys /tmp/docker.asc
```

3. 如果上一步能看到 key 信息, 再转成 gpg 格式

```bash
sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg /tmp/docker.asc  
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

4. 重新更新

```bash
sudo apt update
```

5. 再安装 Docker

```bash
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
