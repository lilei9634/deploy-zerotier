---
name: deploy-zerotier
description: "Deploy ZeroTier on an Alpine Linux server and configure it as a bidirectional LAN gateway. Use this skill whenever the user wants to install ZeroTier on Alpine Linux, set up a ZeroTier gateway or bridge, enable cross-LAN access through ZeroTier, or configure NAT forwarding between ZeroTier and a local network. Also trigger when the user mentions deploying ZeroTier, setting up a VPN gateway on Alpine, or connecting remote LANs through ZeroTier."
---

# Deploy ZeroTier Gateway on Alpine Linux

This skill deploys ZeroTier on an Alpine Linux server and configures it as a **bidirectional LAN gateway**, so that:

1. Remote ZeroTier nodes (and devices on their LANs) can reach the server's local network
2. Devices on the server's local network can reach remote ZeroTier nodes (and their LANs)

## Why this skill exists

Alpine Linux doesn't ship a ZeroTier package in its main or community repos, so the standard `apk add` or the official `install.zerotier.com` script both fail. The workaround is to compile from source. Beyond installation, acting as a LAN gateway requires IP forwarding, iptables NAT rules, TUN device setup, and route configuration in ZeroTier Central — easy to get wrong if done ad-hoc.

## Information to collect

Before starting, gather these from the user:

| Parameter | Example | Notes |
|---|---|---|
| **Server IP** | `192.168.2.6` | SSH target address |
| **SSH user** | `root` | Default: `root`. Must have sudo/root privileges |
| **ZeroTier Network ID** | `93afae5963352110` | 16-character hex string. Ask after install if not provided upfront |

Everything else is **auto-detected** on the server: LAN interface, LAN subnet, ZT interface name, ZT IP, ZT subnet.

## Deployment steps

Run all commands over SSH from the local machine. Use `ssh <user>@<server_ip> "<command>"` throughout.

### Step 1: Verify environment

```bash
ssh <user>@<ip> "cat /etc/os-release && uname -m && ip a && ip route"
```

Confirm it's Alpine Linux and note:
- The primary LAN interface (usually `eth0`) and its subnet (e.g., `192.168.2.0/24`)
- The default gateway
- Architecture (x86_64 expected for compilation)

### Step 2: Enable community repository

Check `/etc/apk/repositories`. If the community line is commented out, uncomment it:

```bash
ssh <user>@<ip> "sed -i 's|^#\(http.*\/community\)|\1|' /etc/apk/repositories"
```

This is needed for some build dependencies.

### Step 3: Install build dependencies

```bash
ssh <user>@<ip> "apk update && apk add build-base linux-headers openssl-dev cargo rust make git nlohmann-json curl bash iptables"
```

This installs the C++/Rust toolchain, headers, and iptables in one pass. The Rust/Cargo dependency is required because ZeroTier's SSO module (`libzeroidc`) is written in Rust.

### Step 4: Compile ZeroTier from source

```bash
ssh <user>@<ip> "cd /tmp && git clone --depth 1 https://github.com/zerotier/ZeroTierOne.git && cd ZeroTierOne && make -j\$(nproc)"
```

This takes a few minutes depending on the server's CPU. Use a **long timeout** (at least 10 minutes / 600000ms) for this command — Rust compilation is slow.

Then install:

```bash
ssh <user>@<ip> "cd /tmp/ZeroTierOne && make install"
```

This places `zerotier-one`, `zerotier-cli`, and `zerotier-idtool` in `/usr/sbin/` with symlinks in `/var/lib/zerotier-one/`.

### Step 5: Create OpenRC service

Write the init script:

```bash
ssh <user>@<ip> 'cat > /etc/init.d/zerotier-one << '\''EOF'\''
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

Enable and start:

```bash
ssh <user>@<ip> "rc-update add zerotier-one default && rc-service zerotier-one start"
```

### Step 6: Ensure TUN device exists

This is a common gotcha on minimal Alpine installs — `/dev/net/tun` may not exist, and without it ZeroTier can't create its virtual network interface.

```bash
ssh <user>@<ip> "modprobe tun && mkdir -p /dev/net && [ -e /dev/net/tun ] || mknod /dev/net/tun c 10 200 && chmod 600 /dev/net/tun && echo tun >> /etc/modules"
```

After creating TUN, restart ZeroTier so it picks up the device:

```bash
ssh <user>@<ip> "rc-service zerotier-one restart"
```

### Step 7: Join ZeroTier network

If you don't have the network ID yet, **ask the user now**.

```bash
ssh <user>@<ip> "zerotier-cli join <network_id>"
```

Verify status:

```bash
ssh <user>@<ip> "zerotier-cli status"
```

You should see `ONLINE` and a 10-character node ID.

**Tell the user to authorize this node** in [ZeroTier Central](https://my.zerotier.com). Provide the node ID so they can find it easily. Then **wait for confirmation** before proceeding.

### Step 8: Verify network connection

After authorization, wait a few seconds and check:

```bash
ssh <user>@<ip> "sleep 3 && zerotier-cli listnetworks && ip a"
```

You should see:
- The network listed with status `OK`
- A new `zt*` interface with an assigned IP

Note the **ZT interface name** (e.g., `ztzlgjw6dt`), **ZT IP** (e.g., `192.168.192.2`), and **ZT subnet** (e.g., `192.168.192.0/24`).

If `listnetworks` is empty or status is not `OK`, the user may not have authorized yet. Ask them to check.

### Step 9: Enable IP forwarding

```bash
ssh <user>@<ip> "grep -q 'net.ipv4.ip_forward=1' /etc/sysctl.conf || echo 'net.ipv4.ip_forward=1' >> /etc/sysctl.conf && sysctl -p"
```

The `grep` guard prevents duplicate entries if run multiple times.

### Step 10: Configure iptables NAT/forwarding

Using the auto-detected values:
- `LAN_IF` = the physical LAN interface (e.g., `eth0`)
- `ZT_IF` = the ZeroTier interface (e.g., `ztzlgjw6dt`)
- `ZT_SUBNET` = ZeroTier network CIDR (e.g., `192.168.192.0/24`)
- `LAN_SUBNET` = local LAN CIDR (e.g., `192.168.2.0/24`)

```bash
ssh <user>@<ip> "ZT_IF=<zt_interface> && LAN_IF=<lan_interface> && \
ZT_SUBNET=<zt_subnet> && LAN_SUBNET=<lan_subnet> && \
iptables -t nat -A POSTROUTING -o \$LAN_IF -s \$ZT_SUBNET -j MASQUERADE && \
iptables -A FORWARD -i \$ZT_IF -o \$LAN_IF -j ACCEPT && \
iptables -A FORWARD -i \$LAN_IF -o \$ZT_IF -m state --state RELATED,ESTABLISHED -j ACCEPT && \
iptables -t nat -A POSTROUTING -o \$ZT_IF -s \$LAN_SUBNET -j MASQUERADE && \
iptables -A FORWARD -i \$LAN_IF -o \$ZT_IF -j ACCEPT && \
iptables -A FORWARD -i \$ZT_IF -o \$LAN_IF -j ACCEPT"
```

What these rules do:
- **First MASQUERADE**: Rewrites source IP for ZT->LAN traffic so LAN devices see packets from the gateway, not from unknown ZT addresses
- **Second MASQUERADE**: Rewrites source IP for LAN->ZT traffic so remote ZT nodes see packets from the gateway's ZT address
- **FORWARD rules**: Allow bidirectional packet flow between the two interfaces

### Step 11: Persist iptables rules

```bash
ssh <user>@<ip> "/etc/init.d/iptables save && rc-update add iptables default"
```

### Step 12: Final verification

```bash
ssh <user>@<ip> "echo '=== ZeroTier ===' && zerotier-cli listnetworks && \
echo '=== IP Forward ===' && cat /proc/sys/net/ipv4/ip_forward && \
echo '=== NAT ===' && iptables -t nat -L POSTROUTING -n -v && \
echo '=== FORWARD ===' && iptables -L FORWARD -n -v && \
echo '=== Interfaces ===' && ip -brief a && \
echo '=== Services ===' && rc-status default"
```

### Step 13: Tell the user what to do next

After deployment, present the user with a summary table and the remaining manual steps:

**Summary:**

| Item | Value |
|---|---|
| ZeroTier version | (from `zerotier-cli status`) |
| Node ID | (10-char hex) |
| ZT Network | (network ID + name) |
| ZT IP | (assigned IP) |
| ZT Interface | (interface name) |
| LAN Interface | (interface name) |
| LAN Subnet | (CIDR) |

**Remaining manual steps:**

1. **Add managed route in ZeroTier Central**: Go to the network settings → Managed Routes → Add:
   - Destination: `<LAN_SUBNET>` (e.g., `192.168.2.0/24`)
   - Via: `<ZT_IP>` (e.g., `192.168.192.2`)

2. **For LAN devices without ZeroTier** that need to reach remote ZT nodes: add a static route on the LAN router pointing the ZT subnet (and any remote LAN subnets) to this server's LAN IP.

3. **If other ZT nodes also serve as LAN gateways**: those nodes need the same IP forwarding + iptables setup, and their LAN subnets need corresponding managed routes in ZeroTier Central.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `listnetworks` empty after join | Node not authorized | User must authorize in ZeroTier Central |
| No `zt*` interface appears | `/dev/net/tun` missing | Run Step 6 (create TUN device), restart ZeroTier |
| `zerotier-one: not found` | Compilation failed | Check `make` output for errors; ensure all deps installed |
| Can't reach LAN from ZT | iptables rules missing or IP forward off | Re-run Steps 9-10, verify with Step 12 |
| LAN devices can't reach ZT | No static route on router | Add route on router: ZT subnet → server LAN IP |

## Cleanup (optional)

After successful deployment, the build dependencies and source code can be removed to save space:

```bash
ssh <user>@<ip> "rm -rf /tmp/ZeroTierOne && apk del build-base linux-headers openssl-dev cargo rust make git nlohmann-json"
```

Keep `curl`, `bash`, and `iptables` — they're useful for ongoing management.
