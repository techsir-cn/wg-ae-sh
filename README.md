# wg-ae-sh
WireGuard AutoUpdate-Endpoint Shell
以下是 ** `wg-ae-spec.md v1.0`**：

---

# wg-ae-spec.md

**WireGuard AutoUpdate-Endpoint 技术规范**  
**说明：一个脚本，它通过强制解析Dns的方法，监测Endpoint域名对应的ip地址变动情况，并且使用变动后的最新ip地址，动态更新对端的Endpoint信息，在不重新启动WireGuard接口的情况下，维持隧道的通信**  
**版本：v1.0**  
**最后更新：2026-02-12**

---

## 1. 目标

解决 WireGuard 的固有限制，这个限制就是--**仅在接口启动时解析 Endpoint 域名一次**，当 Peer 信息中的Endpoint使用的是 DDNS， 且该域名对应的公网 IP 变更后，连接会被迫中断。  

本方案通过定时脚本，执行如下动作：

- 从指定的DNS服务器查询获取域名对应的最新 IP
- 与当前WG接口的当前Peer 运行时的 Endpoint 信息进行比较
- 若不同，则用 `wg set` 动态更新，**而无需重启接口**
- 可以指定权威DNS服务器，规避TTL和递归所需的时间

---

## 2. 核心原则

| 原则 | 说明 |
|------|------|
| **权威 DNS 直连** | 可选指定使用域名对应的权威 DNS 服务器（如 `dns17.hichina.com`），**通过权威DNS而不是公共递归 DNS**（如 223.5.5.5）可以有效规避 TTL 缓存延迟 |
| **IP 版本严格匹配** | 当前 Endpoint 是 IPv4 → 仅查 A 记录；是 IPv6 → 仅查 AAAA 记录；**禁止跨版本替换** |
| **CNAME 透明处理** | 无需特殊逻辑，`dig +short` 自动递归解析 CNAME 至最终 A/AAAA 记录 |
| **配置内聚** | 每个 Peer 的PublicKey、Endpoint的域名:端口、指定 DNS 作为一组配置项，绑定WG接口和Peer，避免全局变量污染 |
| **零外部依赖** | 所有参数写在脚本头部，**不读取外部配置文件** |

---

## 3. 配置格式

脚本头部必须按以下结构定义（支持多接口、多 Peer）：

```sh
# === WG-AE 接口1配置区 ===
INTERFACE="wg0"
LOG_FILE="/etc/wireguard/wg-ae-log"

# Peer 0（必须启用）
PEER_0_PUBLIC_KEY="your public key"
PEER_0_ENDPOINT="your.domain.com:51820"
PEER_0_DNS_SERVER="dns17.hichina.com"

# Peer 1（按需取消注释）
#PEER_1_PUBLIC_KEY="TrMv...WXX0="
#PEER_1_ENDPOINT="node.example.com:51820"
#PEER_1_DNS_SERVER="dns18.hichina.com"
# === WG-AE 接口1配置区结束

# === WG-AE 接口2配置区（复制上一段，改 INTERFACE="wg1"）===
```

### 字段说明

| 字段 | 必填 | 说明 |
|------|------|------|
| `INTERFACE` | 是 | WireGuard 接口名（如 `wg0`） |
| `LOG_FILE` | 是 | **默认硬编码为 `/etc/wireguard/wg-ae-log`**，脚本启动时自动创建目录 |
| `PEER_N_PUBLIC_KEY` | 是 | Peer 公钥（从 `wg0.conf` 复制，**必须与 `wg show` 输出一致**） |
| `PEER_N_ENDPOINT` | 是 | 格式：`域名:端口`（与 `wg0.conf` 中 Endpoint 完全一致） |
| `PEER_N_DNS_SERVER` | 是 | 该域名的**权威 DNS 服务器**（如阿里云分配的 `dns17.hichina.com`）可以在域名服务商的控制台内查看，如果不知道，就填公共DNS服务器，但是可能会受到TTL和递归时间的影响 |

> **命名规则**：  
> - Peer 索引从 `0` 开始连续编号（`PEER_0_*`, `PEER_1_*`, ...）  
> - 多接口场景：复制整个配置区块，仅修改 `INTERFACE`  

---

## 4. 处理逻辑

### 4.1 IP 版本检测

- 从 `wg show $INTERFACE` 提取当前 Peer 的 endpoint（格式：`IP:PORT` 或 `[IPv6]:PORT`）
- **IPv4 判定**：endpoint 字符串不含 `[` 且包含 `.`（如 `192.168.1.1:51820`）
- **IPv6 判定**：endpoint 字符串包含 `[` 和 `]`（如 `[2001::1]:51820`）

### 4.2 DNS 查询规则

| 当前 IP 版本 | DNS 查询命令 |
|--------------|--------------|
| IPv4 | `dig +short A "@$PEER_N_DNS_SERVER" "$domain"` |
| IPv6 | `dig +short AAAA "@$PEER_N_DNS_SERVER" "$domain"` |

> **关键行为**：  
> - `dig +short` 自动处理 CNAME 递归，返回最终 IP  
> - 仅取**第一个结果**（`head -n1`），忽略多记录场景  

### 4.3 更新条件

仅当同时满足以下条件时执行 `wg set`：

1. 新 IP 与当前 IP **字符串不相等**
2. 新 IP 为有效 IP 地址（非空、非错误响应）

---

## 5. 日志规范

每次执行必须记录以下信息到 `LOG_FILE`：

```
[YYYY-MM-DD HH:MM:SS] [wg-ae] Peer N: Domain=xxx | Current=xxx | Resolved=xxx | Status= SAME/DIFFERENT
```

- **Status=DIFFERENT** 时，额外记录更新操作（后续阶段实现）

---

## 6. 兼容性要求

| 环境 | 要求 |
|------|------|
| **Shell** | POSIX `/bin/sh`（兼容 OpenWrt BusyBox 和 Linux bash） |
| **依赖工具** | `wg`, `dig`, `grep`, `awk`, `cut`, `head`（OpenWrt 默认包含） |
| **文件命名** | 仅允许字母、数字、连字符（如 `wg-ae-sh`） |

---

## 7. 版本记录

| 版本 | 日期 | 变更说明 |
|------|------|----------|
| v1.0 | 2026-02-12 | 初始定稿：包含配置格式、DNS 规则、IP 版本匹配、CNAME 处理、多 Peer/接口支持 |

---

## 8. 部署方法（适用于所有系统）

### 步骤 1：准备目录
```sh
mkdir -p /etc/wireguard
```

### 步骤 2：下载并授权脚本
```sh
wget https://raw.githubusercontent.com/yourname/wg-ae/main/wg-ae-sh -O /etc/wireguard/wg-ae-sh && chmod +x /etc/wireguard/wg-ae-sh
```

> 🔔 **注意**：请将 `https://raw.githubusercontent.com/yourname/wg-ae/main/wg-ae-sh` 替换为实际的脚本 URL。

### 步骤 3：配置定时任务
```sh
crontab -e
```
打开的编辑器，类似于vi编辑器，命令格式通用。
进入编辑器，按"i"进入编辑模式，添加以下行：
```cron
*/5 * * * * /etc/wireguard/wg-ae-sh
```
然后按esc进入命令模式，在出现的:后面输入wq后回车，即可保持并退出；
可以使用以下命令来验证是否加入了计划任务：
```sh
crontab -l
```
正确的话，会显示和你输入一样的内容，比如：*/5 * * * * /etc/wireguard/wg-ae-sh

设置cron开机自启动
```sh
/etc/init.d/cron enable
```
手动启动cron
```sh
/etc/init.d/cron start
```
检查是否运行
```sh
/etc/init.d/cron status
```
应该显示
running
则代表cron已经开始运行，每5分钟会自动执行一次

或者使用luci界面来操作cron计划任务


### 步骤 4：安装dig工具
有些openwrt系统没有内置dig工具的，输入以下命令安装
```sh
opkg updat
opkg install bind-dig
```

> ✅ **完成！**  
> - 日志自动写入 `/etc/wireguard/wg-ae-log`
> - 可以输入`cat /etc/wireguard/wg-ae-log或者在/etc/wireguard目录下输入cat wg-ae-log`来查看日志，每5分钟自动执行一次都会有记录
> - 无需区分 Linux/OpenWrt  
> - 无需 systemd/init.d 配置

---

> **此文档为唯一实现依据**。任何脚本生成必须严格遵循本规范，不得擅自修改配置格式或核心逻辑。
