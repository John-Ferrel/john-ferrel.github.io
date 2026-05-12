---
title: wireguard 简易使用
draft: false
tags:
  - vpn
created: 2026-04-16 00:19
modified: 2026-04-16 00:19
---
## 快速新增一个客户端

> 适用场景：  
> 已经在服务器上配置好了 WireGuard，后续只需要**快速新增一个客户端**，生成对应的 `.conf` 文件，发给朋友导入即可。

---

### 功能说明

下面这段脚本会完成这些事情：

1. 生成客户端密钥对
2. 自动把客户端追加到 `/etc/wireguard/wg0.conf`
3. 重启 WireGuard
4. 自动生成客户端配置文件 `/etc/wireguard/<CLIENT_NAME>.conf`
5. 直接打印出配置内容，方便复制

---

### 前置条件

执行前，需要满足：

- 已经安装并配置好 WireGuard
- 服务端配置文件存在：`/etc/wireguard/wg0.conf`
- 需要使用 `root` 执行
- 服务器可以访问公网，用于自动获取公网 IP

---

### 一次性脚本

直接在服务器上执行，修改下面两个变量即可：

- `CLIENT_NAME`
- `CLIENT_IP_LAST`

```bash
cd /etc/wireguard || exit 1

CLIENT_NAME="alice"
CLIENT_IP_LAST="2"
WG_SUBNET_PREFIX="10.66.66"
WG_PORT="51820"
WG_INTERFACE="wg0"

set -euo pipefail

if [ "$(id -u)" -ne 0 ]; then
  echo "请用 root 运行"
  exit 1
fi


if [ ! -f "/etc/wireguard/${WG_INTERFACE}.conf" ]; then
  echo "缺少 WireGuard 配置文件: /etc/wireguard/${WG_INTERFACE}.conf"
  exit 1
fi

CLIENT_IP="${WG_SUBNET_PREFIX}.${CLIENT_IP_LAST}"

if grep -q "#${CLIENT_NAME}" "/etc/wireguard/${WG_INTERFACE}.conf"; then
  echo "用户 ${CLIENT_NAME} 已存在于 ${WG_INTERFACE}.conf 中"
  exit 1
fi

if grep -q "AllowedIPs = ${CLIENT_IP}/32" "/etc/wireguard/${WG_INTERFACE}.conf"; then
  echo "IP ${CLIENT_IP} 已被占用"
  exit 1
fi

wg genkey | tee "${CLIENT_NAME}_private.key" | wg pubkey > "${CLIENT_NAME}_public.key"
chmod 600 "${CLIENT_NAME}_private.key" "${CLIENT_NAME}_public.key"

cat <<EOF >> "/etc/wireguard/${WG_INTERFACE}.conf"

[Peer]
#${CLIENT_NAME}
PublicKey = $(cat "${CLIENT_NAME}_public.key")
AllowedIPs = ${CLIENT_IP}/32
EOF

systemctl restart "wg-quick@${WG_INTERFACE}"

SERVER_PUBLIC_IP="$(curl -4 -fsSL ifconfig.co)"
SERVER_PUBLIC_KEY="$(wg show ${WG_INTERFACE} public-key)"

cat <<EOF > "/etc/wireguard/${CLIENT_NAME}.conf"
[Interface]
PrivateKey = $(cat "${CLIENT_NAME}_private.key")
Address = ${CLIENT_IP}/24
DNS = 8.8.8.8

[Peer]
PublicKey = ${SERVER_PUBLIC_KEY}
Endpoint = ${SERVER_PUBLIC_IP}:${WG_PORT}
AllowedIPs = ${WG_SUBNET_PREFIX}.0/24
PersistentKeepalive = 25
EOF

chmod 600 "/etc/wireguard/${CLIENT_NAME}.conf"

echo
echo "===== 已创建客户端: ${CLIENT_NAME} ====="
echo "客户端 IP: ${CLIENT_IP}"
echo "客户端配置文件: /etc/wireguard/${CLIENT_NAME}.conf"
echo
echo "===== 配置内容如下 ====="
cat "/etc/wireguard/${CLIENT_NAME}.conf"
```

---

### 使用示例

比如新增一个用户 `alice`，给她分配地址 `10.66.66.2`：

```bash
CLIENT_NAME="alice"
CLIENT_IP_LAST="2"
```

比如再新增一个 `bob`，给他分配地址 `10.66.66.3`：

```bash
CLIENT_NAME="bob"
CLIENT_IP_LAST="3"
```

---

### 生成后的结果

执行成功后，会得到两个结果：

#### 1. 服务端 `wg0.conf` 自动追加一个 Peer

类似这样：

```ini
[Peer]
#alice
PublicKey = xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx=
AllowedIPs = 10.66.66.2/32
```

#### 2. 自动生成客户端配置文件

路径：

```bash
/etc/wireguard/alice.conf
```

内容类似：

```ini
[Interface]
PrivateKey = xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx=
Address = 10.66.66.2/24
DNS = 8.8.8.8

[Peer]
PublicKey = xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx=
Endpoint = your.server.ip:51820
AllowedIPs = 10.66.66.0/24
PersistentKeepalive = 25
```

把这个文件内容发给客户端，导入 WireGuard 即可。

---

### 参数说明

#### `CLIENT_NAME`

客户端名称，用于：

- 密钥文件名
- 配置文件名
- 服务端 Peer 注释名

例如：

```bash
CLIENT_NAME="alice"
```

会生成：

- `alice_private.key`
- `alice_public.key`
- `alice.conf`

---

#### `CLIENT_IP_LAST`

客户端使用的最后一段 IP。

例如：

```bash
WG_SUBNET_PREFIX="10.66.66"
CLIENT_IP_LAST="2"
```

最终客户端 IP 就是：

```text
10.66.66.2
```

---

#### `WG_SUBNET_PREFIX`

WireGuard 的网段前缀。

例如：

```bash
WG_SUBNET_PREFIX="10.66.66"
```

表示整个 VPN 网段是：

```text
10.66.66.0/24
```

---

#### `WG_PORT`

WireGuard 服务端监听端口。

默认：

```bash
WG_PORT="51820"
```

---

#### `WG_INTERFACE`

WireGuard 接口名。

默认：

```bash
WG_INTERFACE="wg0"
```

---

### 注意事项

#### 1. 不要重复使用同一个 IP

每个客户端都应该使用唯一的 VPN 地址。

例如：

- `alice` → `10.66.66.2`
- `bob` → `10.66.66.3`
- `charlie` → `10.66.66.4`

---

#### 2. 私钥不要泄露

脚本会生成：

- `<CLIENT_NAME>_private.key`
- `<CLIENT_NAME>.conf`

其中都包含客户端私钥。

这些内容：

- 不要公开发到群里
- 不要提交到 Git
- 不要放到公开文档里

---

#### 3. `AllowedIPs` 不是全局代理

这里生成的客户端配置：

```ini
AllowedIPs = 10.66.66.0/24
```

表示只有访问 VPN 网段的流量才会走 WireGuard。

也就是说，这更像是：

- 访问内网服务
- 访问 VPN 内的 SillyTavern
- 不改变平时上网默认出口

这正适合“访问私有服务器服务”的场景。

---

#### 4. 自动获取公网 IP 依赖外部服务

脚本使用：

```bash
curl -4 -fsSL ifconfig.co
```

来自动获取服务器公网 IP。

如果服务器无法访问这个地址，可以手动替换成固定公网 IP：

```bash
SERVER_PUBLIC_IP="1.2.3.4"
```

---

### 推荐做法：保存成脚本

如果以后经常要加人，建议保存为脚本文件，例如：

```bash
/usr/local/bin/wg-add-client.sh
```

然后做成带参数的版本。

---

**创建脚本文件**

执行: 

```bash
sudo nano /usr/local/bin/wg-add-client.sh
```


**参数化版本**: 

把下面整段内容原样粘进去

```bash
#!/usr/bin/env bash
set -euo pipefail

if [ "$(id -u)" -ne 0 ]; then
  echo "请用 root 运行"
  exit 1
fi

if [ "$#" -ne 2 ]; then
  echo "用法: $0 <CLIENT_NAME> <CLIENT_IP_LAST>"
  echo "示例: $0 alice 2"
  exit 1
fi

CLIENT_NAME="$1"
CLIENT_IP_LAST="$2"

cd /etc/wireguard || exit 1

WG_SUBNET_PREFIX="10.66.66"
WG_PORT="51820"
WG_INTERFACE="wg0"

if [ ! -f "/etc/wireguard/${WG_INTERFACE}.conf" ]; then
  echo "缺少 WireGuard 配置文件: /etc/wireguard/${WG_INTERFACE}.conf"
  exit 1
fi

CLIENT_IP="${WG_SUBNET_PREFIX}.${CLIENT_IP_LAST}"

if grep -q "#${CLIENT_NAME}" "/etc/wireguard/${WG_INTERFACE}.conf"; then
  echo "用户 ${CLIENT_NAME} 已存在于 ${WG_INTERFACE}.conf 中"
  exit 1
fi

if grep -q "AllowedIPs = ${CLIENT_IP}/32" "/etc/wireguard/${WG_INTERFACE}.conf"; then
  echo "IP ${CLIENT_IP} 已被占用"
  exit 1
fi

wg genkey | tee "${CLIENT_NAME}_private.key" | wg pubkey > "${CLIENT_NAME}_public.key"
chmod 600 "${CLIENT_NAME}_private.key" "${CLIENT_NAME}_public.key"

cat <<EOF >> "/etc/wireguard/${WG_INTERFACE}.conf"

[Peer]
#${CLIENT_NAME}
PublicKey = $(cat "${CLIENT_NAME}_public.key")
AllowedIPs = ${CLIENT_IP}/32
EOF

systemctl restart "wg-quick@${WG_INTERFACE}"

SERVER_PUBLIC_KEY="$(wg show ${WG_INTERFACE} public-key)"
SERVER_PUBLIC_IP="$(curl -4 -fsSL ifconfig.co)"

cat <<EOF > "/etc/wireguard/${CLIENT_NAME}.conf"
[Interface]
PrivateKey = $(cat "${CLIENT_NAME}_private.key")
Address = ${CLIENT_IP}/24
DNS = 8.8.8.8

[Peer]
PublicKey = ${SERVER_PUBLIC_KEY}
Endpoint = ${SERVER_PUBLIC_IP}:${WG_PORT}
AllowedIPs = ${WG_SUBNET_PREFIX}.0/24
PersistentKeepalive = 25
EOF

chmod 600 "/etc/wireguard/${CLIENT_NAME}.conf"

echo
echo "===== 已创建客户端: ${CLIENT_NAME} ====="
echo "客户端 IP: ${CLIENT_IP}"
echo "客户端配置文件: /etc/wireguard/${CLIENT_NAME}.conf"
echo
cat "/etc/wireguard/${CLIENT_NAME}.conf"
```

---

**保存并退出**:

在 `nano` 里：

- 按 `Ctrl + O` 保存
- 回车确认
- 按 `Ctrl + X` 退出

### 使用方式

**给脚本加执行权限**：

```bash
sudo chmod +x /usr/local/bin/wg-add-client.sh
```

**测试可用性**:

执行: 
```bash
sudo wg-add-client.sh
```

输出:

```bash
用法: /usr/local/bin/wg-add-client.sh <CLIENT_NAME> <CLIENT_IP_LAST>
示例: /usr/local/bin/wg-add-client.sh alice 2
```

**正式使用**: 

执行：

```bash
sudo wg-add-client.sh alice 2
```

再执行：

```bash
sudo wg-add-client.sh bob 3
```

**导出配置文件**:

```bash
sudo cat /etc/wireguard/alice.conf
```

脚本会把密钥相关文件留在 `/etc/wireguard`目录下。

可以直接复制内容发给对方，或下载该文件后导入 WireGuard 客户端。

---

