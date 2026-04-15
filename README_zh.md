[English](README.md) | **中文**

# deploy-zerotier

一个 [Claude Code](https://claude.ai/claude-code) 技能，用于在 Alpine Linux 上自动部署 ZeroTier 并将其配置为**双向局域网网关**。

## 功能

```
你的局域网 (如 192.168.2.0/24)
       │
  Alpine 服务器 (ZeroTier 网关)
       │  zt 接口: ZeroTier 虚拟网络
       │
  远程 ZeroTier 节点 + 它们的局域网
```

- **自带预编译二进制文件**，适用于 x86_64 Alpine — 跳过约 10 分钟的编译步骤
- **预编译不可用时自动回退到源码编译**（不同架构等情况）
- **创建 OpenRC 服务**，开机自启
- **配置双向 NAT/转发**，让两端都能访问对方的局域网
- **所有设置持久化**，重启后依然生效（iptables、IP 转发、TUN 模块）

## 前提条件

- 有 SSH root 访问权限的 Alpine Linux 服务器
- 一个 ZeroTier 网络（在 [my.zerotier.com](https://my.zerotier.com) 创建）

## 使用方法

### 在 Claude Code 中使用

将 `deploy-zerotier/` 文件夹（包含 `SKILL.md`）放入你的 Claude Code 技能目录，然后对 Claude 说：

> "帮我在 Alpine 服务器 192.168.2.6 上部署 ZeroTier"

Claude 会引导你完成整个部署流程，期间会询问你的 ZeroTier 网络 ID，并在需要授权节点时暂停等待。

### 手动操作

你也可以直接按照 [SKILL_zh.md](SKILL_zh.md) 中的步骤手动执行。

## 安装的组件

| 组件 | 详情 |
|---|---|
| ZeroTier One | 预编译二进制文件 (x86_64 musl) 或从[源码](https://github.com/zerotier/ZeroTierOne)编译，安装到 `/usr/sbin/` |
| OpenRC 服务 | `/etc/init.d/zerotier-one`，开机自启 |
| IP 转发 | `/etc/sysctl.conf` 中设置 `net.ipv4.ip_forward=1` |
| iptables 规则 | 双向 MASQUERADE + FORWARD，通过 `/etc/init.d/iptables save` 持久化 |
| TUN 设备 | `/dev/net/tun`，`tun` 模块开机自动加载 |

## 部署后还需要做

1. **在 ZeroTier Central 添加托管路由**：你的局域网网段 → 服务器的 ZT IP
2. **在局域网路由器上添加静态路由**（让没装 ZeroTier 的设备也能访问远程网络）
3. 如果远程 ZT 节点也需要做局域网网关，需要在那些节点上重复相同的配置

## 许可证

MIT
