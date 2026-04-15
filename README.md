**English** | [中文](README_zh.md)

# deploy-zerotier

A [Claude Code](https://claude.ai/claude-code) skill that automates ZeroTier deployment on Alpine Linux and configures it as a **bidirectional LAN gateway**.

## What it does

```
Your LAN (e.g. 192.168.2.0/24)
       │
  Alpine Server (ZeroTier Gateway)
       │  zt interface: ZeroTier overlay
       │
  Remote ZeroTier nodes + their LANs
```

- **Ships a pre-compiled binary** for x86_64 Alpine — skips the ~10 min compilation step
- **Falls back to source compilation** if the binary doesn't work (different arch, etc.)
- **Creates OpenRC service** for auto-start
- **Configures bidirectional NAT/forwarding** so both sides can reach each other's LAN
- **Persists all settings** across reboots (iptables, IP forwarding, TUN module)

## Requirements

- Alpine Linux server with SSH access (root)
- A ZeroTier network (create one at [my.zerotier.com](https://my.zerotier.com))

## Usage

### In Claude Code

Place the `deploy-zerotier/` folder (containing `SKILL.md`) into your Claude Code skills directory, then ask Claude:

> "Help me deploy ZeroTier on my Alpine server 192.168.2.6"

Claude will walk through the full deployment, asking for your ZeroTier network ID and pausing for you to authorize the node.

### Manual

You can also follow the step-by-step instructions in [SKILL.md](SKILL.md) directly.

## What gets installed

| Component | Details |
|---|---|
| ZeroTier One | Pre-compiled binary (x86_64 musl) or compiled from [source](https://github.com/zerotier/ZeroTierOne), installed to `/usr/sbin/` |
| OpenRC service | `/etc/init.d/zerotier-one`, auto-start on boot |
| IP forwarding | `net.ipv4.ip_forward=1` in `/etc/sysctl.conf` |
| iptables rules | Bidirectional MASQUERADE + FORWARD, persisted via `/etc/init.d/iptables save` |
| TUN device | `/dev/net/tun`, `tun` module loaded on boot |

## After deployment

You still need to:

1. **Add a managed route** in ZeroTier Central: your LAN subnet → server's ZT IP
2. **Add static routes** on your LAN router (for non-ZeroTier devices to reach remote networks)
3. If remote ZT nodes also serve as LAN gateways, repeat the same setup on those nodes

## License

MIT
