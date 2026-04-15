[English](SKILL.md) | **中文**

---
name: deploy-zerotier
description: "在 Alpine Linux 服务器上部署 ZeroTier 并配置为双向局域网网关。当用户想在 Alpine Linux 上安装 ZeroTier、搭建 ZeroTier 网关/网桥、实现跨局域网互访、或配置 ZeroTier 与局域网之间的 NAT 转发时，使用此技能。"
---

# 在 Alpine Linux 上部署 ZeroTier 网关

本技能在 Alpine Linux 服务器上部署 ZeroTier，并将其配置为**双向局域网网关**，实现：

1. 远程 ZeroTier 节点（及其局域网内的设备）可以访问服务器所在的局域网
2. 服务器所在局域网的设备可以访问远程 ZeroTier 节点（及其局域网）

## 为什么需要这个技能

Alpine Linux 的 main 和 community 仓库都没有 ZeroTier 软件包，标准的 `apk add` 和官方 `install.zerotier.com` 脚本都无法使用。本技能在 `bin/zerotier-one` 中附带了一个**预编译的 x86_64 musl 二进制文件**（在 Alpine 3.23 上构建），可跳过漫长的编译步骤。如果预编译文件在目标服务器上无法运行（架构不匹配、musl 版本不兼容等），则回退到源码编译。此外，充当局域网网关还需要 IP 转发、iptables NAT 规则、TUN 设备配置以及 ZeroTier Central 中的路由设置——手动操作容易出错。

## 需要收集的信息

开始前，从用户处获取以下信息：

| 参数 | 示例 | 说明 |
|---|---|---|
| **服务器 IP** | `192.168.2.6` | SSH 连接地址 |
| **SSH 用户** | `root` | 默认 `root`，需要 sudo/root 权限 |
| **ZeroTier 网络 ID** | `93afae5963352110` | 16 位十六进制字符串，安装后再问也可以 |

其他信息在服务器上**自动检测**：局域网接口、局域网网段、ZT 接口名、ZT IP、ZT 网段。

## 部署步骤

所有命令通过 SSH 从本地机器执行，格式为 `ssh <用户>@<服务器IP> "<命令>"`。

### 步骤 1：检查环境

```bash
ssh <用户>@<IP> "cat /etc/os-release && uname -m && ip a && ip route"
```

确认是 Alpine Linux，记下：
- 主网络接口（通常是 `eth0`）及其网段（如 `192.168.2.0/24`）
- 默认网关
- CPU 架构（`x86_64` 表示预编译二进制文件可用；其他架构需要源码编译）

### 步骤 2：安装 ZeroTier 二进制文件（优先预编译 → 回退源码编译）

本技能在 `bin/zerotier-one` 中附带了预编译二进制文件（Alpine 3.23, x86_64, musl 链接）。优先尝试使用它——比源码编译快约 10 分钟。

**策略：** 上传预编译二进制 → 测试 → 能用就创建符号链接继续。不能用就回退到源码编译。

#### 方案 A：使用预编译二进制（快速路径）

首先找到本技能目录下 `bin/zerotier-one` 的绝对路径，然后上传并测试：

```bash
scp <技能路径>/bin/zerotier-one <用户>@<IP>:/usr/sbin/zerotier-one
ssh <用户>@<IP> "chmod +x /usr/sbin/zerotier-one && /usr/sbin/zerotier-one --version"
```

如果 `--version` 命令输出了版本号（如 `ZeroTier One version 1.16.1`），说明二进制文件可用。创建符号链接和数据目录：

```bash
ssh <用户>@<IP> "ln -sf /usr/sbin/zerotier-one /usr/sbin/zerotier-cli && \
ln -sf /usr/sbin/zerotier-one /usr/sbin/zerotier-idtool && \
mkdir -p /var/lib/zerotier-one && \
ln -sf /usr/sbin/zerotier-one /var/lib/zerotier-one/zerotier-one && \
ln -sf /usr/sbin/zerotier-one /var/lib/zerotier-one/zerotier-cli && \
ln -sf /usr/sbin/zerotier-one /var/lib/zerotier-one/zerotier-idtool"
```

安装运行时依赖（iptables 后面要用到）：

```bash
ssh <用户>@<IP> "apk update && apk add iptables"
```

然后**跳到步骤 3**。

#### 方案 B：源码编译（回退路径）

如果预编译二进制无法使用（架构不对、动态库问题等），回退到源码编译。

启用 community 仓库（如需要）：

```bash
ssh <用户>@<IP> "sed -i 's|^#\(http.*\/community\)|\1|' /etc/apk/repositories"
```

安装编译依赖：

```bash
ssh <用户>@<IP> "apk update && apk add build-base linux-headers openssl-dev cargo rust make git nlohmann-json curl bash iptables"
```

克隆并编译（使用**较长的超时时间**，至少 10 分钟 / 600000ms——Rust 编译很慢）：

```bash
ssh <用户>@<IP> "cd /tmp && git clone --depth 1 https://github.com/zerotier/ZeroTierOne.git && cd ZeroTierOne && make -j\$(nproc)"
```

安装：

```bash
ssh <用户>@<IP> "cd /tmp/ZeroTierOne && make install"
```

这会将 `zerotier-one`、`zerotier-cli` 和 `zerotier-idtool` 安装到 `/usr/sbin/`，并在 `/var/lib/zerotier-one/` 创建符号链接。

### 步骤 3：创建 OpenRC 服务

写入 init 脚本：

```bash
ssh <用户>@<IP> 'cat > /etc/init.d/zerotier-one << '\''EOF'\''
#!/sbin/openrc-run

name="zerotier-one"
description="ZeroTier One"
command="/usr/sbin/zerotier-one"
command_background="yes"
pidfile="/var/lib/zerotier-one/zerotier-one.pid"

depend() {
    need net
    after firewall
}
EOF
chmod +x /etc/init.d/zerotier-one'
```

启用并启动：

```bash
ssh <用户>@<IP> "rc-update add zerotier-one default && rc-service zerotier-one start"
```

### 步骤 4：确保 TUN 设备存在

这是精简版 Alpine 安装时的常见问题——`/dev/net/tun` 可能不存在，没有它 ZeroTier 就无法创建虚拟网络接口。

```bash
ssh <用户>@<IP> "modprobe tun && mkdir -p /dev/net && [ -e /dev/net/tun ] || mknod /dev/net/tun c 10 200 && chmod 600 /dev/net/tun && echo tun >> /etc/modules"
```

创建 TUN 后，重启 ZeroTier 以使其识别设备：

```bash
ssh <用户>@<IP> "rc-service zerotier-one restart"
```

### 步骤 5：加入 ZeroTier 网络

如果还没有网络 ID，**现在向用户询问**。

```bash
ssh <用户>@<IP> "zerotier-cli join <网络ID>"
```

验证状态：

```bash
ssh <用户>@<IP> "zerotier-cli status"
```

应该看到 `ONLINE` 和一个 10 位的节点 ID。

**告知用户需要在 [ZeroTier Central](https://my.zerotier.com) 授权此节点**。提供节点 ID 以便用户查找。然后**等待用户确认**再继续。

### 步骤 6：验证网络连接

用户授权后，等几秒检查：

```bash
ssh <用户>@<IP> "sleep 3 && zerotier-cli listnetworks && ip a"
```

应该看到：
- 网络列表中显示状态为 `OK`
- 一个新的 `zt*` 接口，带有分配的 IP

记下 **ZT 接口名**（如 `ztzlgjw6dt`）、**ZT IP**（如 `192.168.192.2`）和 **ZT 网段**（如 `192.168.192.0/24`）。

如果 `listnetworks` 为空或状态不是 `OK`，可能用户还没授权，提醒用户检查。

### 步骤 7：启用 IP 转发

```bash
ssh <用户>@<IP> "grep -q 'net.ipv4.ip_forward=1' /etc/sysctl.conf || echo 'net.ipv4.ip_forward=1' >> /etc/sysctl.conf && sysctl -p"
```

`grep` 防止多次执行时重复写入。

### 步骤 8：配置 iptables NAT/转发

使用自动检测到的值：
- `LAN_IF` = 物理局域网接口（如 `eth0`）
- `ZT_IF` = ZeroTier 接口（如 `ztzlgjw6dt`）
- `ZT_SUBNET` = ZeroTier 网络 CIDR（如 `192.168.192.0/24`）
- `LAN_SUBNET` = 本地局域网 CIDR（如 `192.168.2.0/24`）

```bash
ssh <用户>@<IP> "ZT_IF=<zt接口> && LAN_IF=<lan接口> && \
ZT_SUBNET=<zt网段> && LAN_SUBNET=<lan网段> && \
iptables -t nat -A POSTROUTING -o \$LAN_IF -s \$ZT_SUBNET -j MASQUERADE && \
iptables -A FORWARD -i \$ZT_IF -o \$LAN_IF -j ACCEPT && \
iptables -A FORWARD -i \$LAN_IF -o \$ZT_IF -m state --state RELATED,ESTABLISHED -j ACCEPT && \
iptables -t nat -A POSTROUTING -o \$ZT_IF -s \$LAN_SUBNET -j MASQUERADE && \
iptables -A FORWARD -i \$LAN_IF -o \$ZT_IF -j ACCEPT && \
iptables -A FORWARD -i \$ZT_IF -o \$LAN_IF -j ACCEPT"
```

这些规则的作用：
- **第一个 MASQUERADE**：将 ZT→LAN 流量的源 IP 改写为网关地址，让局域网设备不会收到未知的 ZT 地址
- **第二个 MASQUERADE**：将 LAN→ZT 流量的源 IP 改写为网关的 ZT 地址，让远程 ZT 节点能正确回复
- **FORWARD 规则**：允许两个接口之间的双向数据包转发

### 步骤 9：持久化 iptables 规则

```bash
ssh <用户>@<IP> "/etc/init.d/iptables save && rc-update add iptables default"
```

### 步骤 10：最终验证

```bash
ssh <用户>@<IP> "echo '=== ZeroTier ===' && zerotier-cli listnetworks && \
echo '=== IP 转发 ===' && cat /proc/sys/net/ipv4/ip_forward && \
echo '=== NAT ===' && iptables -t nat -L POSTROUTING -n -v && \
echo '=== FORWARD ===' && iptables -L FORWARD -n -v && \
echo '=== 接口 ===' && ip -brief a && \
echo '=== 服务 ===' && rc-status default"
```

### 步骤 11：告知用户后续操作

部署完成后，向用户展示汇总表和剩余的手动步骤：

**汇总：**

| 项目 | 值 |
|---|---|
| ZeroTier 版本 | （来自 `zerotier-cli status`） |
| 节点 ID | （10 位十六进制） |
| ZT 网络 | （网络 ID + 名称） |
| ZT IP | （分配的 IP） |
| ZT 接口 | （接口名） |
| 局域网接口 | （接口名） |
| 局域网网段 | （CIDR） |

**剩余手动步骤：**

1. **在 ZeroTier Central 添加托管路由**：进入网络设置 → Managed Routes → 添加：
   - Destination（目标）：`<局域网网段>`（如 `192.168.2.0/24`）
   - Via（网关）：`<ZT IP>`（如 `192.168.192.2`）

2. **局域网中没有安装 ZeroTier 的设备**如需访问远程 ZT 节点：在局域网路由器上添加静态路由，将 ZT 网段（及远程局域网网段）指向服务器的局域网 IP。

3. **如果其他 ZT 节点也需要做局域网网关**：这些节点需要相同的 IP 转发 + iptables 配置，并在 ZeroTier Central 中添加对应的托管路由。

## 故障排查

| 症状 | 原因 | 解决方法 |
|---|---|---|
| `listnetworks` 加入后为空 | 节点未授权 | 用户须在 ZeroTier Central 授权 |
| 没有 `zt*` 接口出现 | `/dev/net/tun` 不存在 | 执行步骤 4（创建 TUN 设备），重启 ZeroTier |
| `zerotier-one: not found` | 二进制上传失败且编译也失败 | 尝试步骤 2 中的方案 B；检查 `make` 输出的错误 |
| ZT 无法访问局域网 | iptables 规则缺失或 IP 转发未开启 | 重新执行步骤 7-8，用步骤 10 验证 |
| 局域网设备无法访问 ZT | 路由器上无静态路由 | 在路由器上添加路由：ZT 网段 → 服务器局域网 IP |

## 清理（可选）

部署成功后，可以删除编译依赖和源码以节省空间：

```bash
ssh <用户>@<IP> "rm -rf /tmp/ZeroTierOne && apk del build-base linux-headers openssl-dev cargo rust make git nlohmann-json"
```

保留 `curl`、`bash` 和 `iptables`——日常管理还需要它们。
