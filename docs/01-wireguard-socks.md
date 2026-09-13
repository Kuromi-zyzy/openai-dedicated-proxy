# 01 · 主力线路：WireGuard 隧道 + SOCKS5

> 目标：任意 VPS（Ubuntu 22.04+ / Debian 12，x86_64，需有 TUN 设备）上跑 WireGuard，客户端两条接法（mihomo 用户态 / Windows 内核态 + 服务器 SOCKS5），出口固定为 VPS 公网 IP。
> 本文讲原理与细节；**一步步执行的 runbook 见 [AGENT_RUNBOOK.md](../AGENT_RUNBOOK.md) §4**。
> `<占位符>` 按你的环境替换；`10.66.66.0/24` 隧道网段可沿用（RFC1918 私网，自选网段亦可）。

## 0. 为什么这条线是主力

- 配置量最小，两端都是内核级实现（服务器内核转发；Windows 可用官方内核态客户端），延迟最低；
- `AllowedIPs` 分流：只把 OpenAI 生态流量引进隧道，其它流量不消耗 VPS 出站流量；
- 弱点：UDP 特征明显，端口被定向阻断时需要换端口——好在判别+迁移流程成熟（见 [坑 1](05-troubleshooting.md)）。

防火墙要求：放行 **UDP `<WG_PORT>`（默认 52800，勿用 51820）** + TCP 22。

## 1. 安装

```bash
apt update && apt install -y wireguard microsocks
# 内核转发（必需，否则隧道流量出不去）
echo 'net.ipv4.ip_forward = 1' > /etc/sysctl.d/99-wg-forward.conf
sysctl --system
```

密钥生成（服务端一对 + 每客户端各一对）：

```bash
wg genkey | tee server_private.key | wg pubkey > server_public.key
wg genkey | tee client2_private.key | wg pubkey > client2_public.key   # mihomo 用户态
wg genkey | tee client3_private.key | wg pubkey > client3_public.key   # Windows 内核态（可选）
```

## 2. wg0.conf 与三大关键点

`/etc/wireguard/wg0.conf`（root:root 600），模板见 [examples/wg0.conf.example](../examples/wg0.conf.example)：

```ini
[Interface]
Address = 10.66.66.1/24
MTU = 1380
ListenPort = 52800
PrivateKey = <服务器私钥>

# ① 转发放行：wg0 ↔ 出口网卡
PostUp = iptables -I FORWARD 1 -i %i -o <NIC> -s 10.66.66.0/24 -j ACCEPT
PostUp = iptables -I FORWARD 1 -i <NIC> -o %i -d 10.66.66.0/24 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
# ② SNAT：写死【挂了公网 IP 的那个内网 IP】
PostUp = iptables -t nat -I POSTROUTING 1 -s 10.66.66.0/24 -o <NIC> -j SNAT --to-source <MAIN_PRIVATE_IP>
# ③ MSS clamp：修 OpenAI(Cloudflare) 回包 mss 1400 > 隧道 MTU 1380 的分片丢包
PostUp = iptables -t mangle -I FORWARD 1 -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu

PostDown = iptables -D FORWARD -i %i -o <NIC> -s 10.66.66.0/24 -j ACCEPT
PostDown = iptables -D FORWARD -i <NIC> -o %i -d 10.66.66.0/24 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
PostDown = iptables -t nat -D POSTROUTING -s 10.66.66.0/24 -o <NIC> -j SNAT --to-source <MAIN_PRIVATE_IP>
PostDown = iptables -t mangle -D FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu

[Peer]
PublicKey = <client2 公钥>
AllowedIPs = 10.66.66.2/32

[Peer]
PublicKey = <client3 公钥>
AllowedIPs = 10.66.66.3/32
```

### 关键点 ①②：为什么 SNAT 写死、不用 MASQUERADE

部分云平台给网卡挂**多个内网 IP**，其中辅助 IP 没有公网映射。`MASQUERADE` 按路由自动选源地址，可能选中辅助 IP——症状是**握手全部正常、客户端发送字节正常上涨、接收几乎不涨**（"有去无回"）。实测数据：误选辅助 IP 时收 386KiB / 回 8KiB；改 `SNAT --to-source <主内网IP>` 后立即恢复。

查主 IP：`ip -4 route show default` 找默认网卡，`ip -4 addr show <NIC>` 里带 `scope global` 的第一个地址；拿不准就看云控制台网卡页的"主 IP"。

### 关键点 ③：MTU 1380 + MSS clamp

OpenAI 走 Cloudflare，TLS 握手 SYN-ACK 携带 `mss 1400`；隧道 MTU 1420 时可用载荷仅 1340，服务器转发时需要分片，在移动网络表现为**间歇性 SACK 重传、dial 超时、stream disconnected**。两步修复：

- 隧道 `MTU = 1380`；
- mangle 表对转发 SYN 做 `TCPMSS --clamp-mss-to-pmtu`（即上面 PostUp ③），让对端协商到 1340。修复后所有 SYN 的 mss=1340，分片丢包消失。

### 端口：52800，绝不用 51820

WireGuard 默认端口 51820 在国内链路上存在**定向阻断**（握手能到服务器、服务器正常回包、客户端永远收不到；换 52800 立即恢复）。判别方法与迁移流程见 [排坑手册坑 1](05-troubleshooting.md)。

## 3. 启动与防火墙

```bash
chmod 600 /etc/wireguard/wg0.conf
systemctl enable --now wg-quick@wg0
wg show    # 等客户端连上后应有握手记录
```

防火墙增量放行（**只加这一条，绝不关防火墙**）：

```bash
# ufw（Ubuntu 常见）：
ufw allow 52800/udp comment 'WireGuard (never use 51820)'
# firewalld（RHEL 系）：
firewall-cmd --permanent --add-port=52800/udp && firewall-cmd --reload
```

另外记得在**云控制台的安全组/防火墙**放行 UDP 52800（这是平台层，与机器内防火墙是两回事）。

> ⚠️ 不要放行 1080——SOCKS5 只在隧道内网可用（见下）。

## 4. SOCKS5 出口：microsocks（给内核态 WG 客户端用）

mihomo 用户态 WG 直接连 wg0 出站即可；Windows 官方内核态客户端只建立隧道不提供 SOCKS，需要在服务器加一个**只监听隧道内网**的 SOCKS5。

`/etc/microsocks-wg.env`（root 600，勿提交仓库）：

```
MSPASS=<强随机密码>
```

`/etc/systemd/system/microsocks-wg.service`（完整版见 [examples/microsocks-wg.service.example](../examples/microsocks-wg.service.example)）：

```ini
[Unit]
Description=microsocks SOCKS5 on WireGuard tunnel (10.66.66.1:1080)
After=network-online.target wg-quick@wg0.service
Wants=wg-quick@wg0.service

[Service]
Type=simple
EnvironmentFile=/etc/microsocks-wg.env
ExecStart=/bin/sh -c 'exec /usr/bin/microsocks -i 10.66.66.1 -p 1080 -u wgproxy -P "$MSPASS"'
Restart=always
RestartSec=2
NoNewPrivileges=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload && systemctl enable --now microsocks-wg
ss -tlnp | grep 1080   # 必须显示 10.66.66.1:1080，绝不能是 0.0.0.0:1080
```

> microsocks 只支持 TCP（无 UDP ASSOCIATE），对 Codex/ChatGPT/API 这类 TCP 长连接无影响。

### 安全边界（三层，缺一不可）

1. 流量在 WG 加密隧道内（SOCKS 明文，但不出隧道）；
2. 只绑 `10.66.66.1` 内网地址——公网 IP:1080 实测不可达；
3. SOCKS 用户名 + 密码。

**历史教训**：早期方案曾把 SOCKS5 直接暴露公网（防火墙未启用 + 平台防火墙全放），31 个 IDC 扫描器长期挂着连接。迁入隧道后绝不可回退。

## 5. 客户端接入（两种接法并存）

### 接法 A：mihomo 用户态 WG（Clash Verge 内完成，开箱即用）

```yaml
- name: VPS-WG
  type: wireguard
  server: <VPS_IP>
  port: 52800
  ip: 10.66.66.2
  private-key: <client2 私钥>
  public-key: <服务器公钥>
  udp: true
  mtu: 1380
  persistent-keepalive: 25
  remote-dns-resolve: true
  dns: [1.1.1.1]
```

### 接法 B：Windows 内核态 WG + 服务器 SOCKS5（稳定性最高）

同窗对照实测（各 60 轮交替探测）：用户态 48/60 vs 内核态 60/60（当时端口正被渐进干扰，内核栈对丢包更耐受；端口健康后两者都可用，内核态仍建议保留为备用路径）。mihomo 里它就是一个普通 socks5 节点：

```yaml
- name: VPS-WG-SOCKS
  type: socks5
  server: 10.66.66.1
  port: 1080
  username: wgproxy
  password: <MSPASS>
  udp: false
```

Windows 内核态客户端配置（客户端 B，用 client3 的密钥）：

```ini
[Interface]
PrivateKey = <client3 私钥>
Address = 10.66.66.3/24

[Peer]
PublicKey = <服务器公钥>
Endpoint = <VPS_IP>:52800
AllowedIPs = 10.66.66.0/24     # split tunnel：只路由隧道网段，不抢全局
PersistentKeepalive = 25
```

以系统服务安装：

```powershell
# 管理员 PowerShell
wireguard.exe /installtunnelservice C:\ProgramData\WireGuard\vps-proxy.conf
# 改配置后必须重启服务（不会热加载）
Restart-Service -Name 'WireGuardTunnel$vps-proxy'
```

> ⚠️ 命令行装的隧道**不出现在 WireGuard GUI 列表里**（GUI 只列 `%ProgramFiles%\WireGuard\Data\Configurations\`），GUI 显示空白是正常的；配置文件直接改 `C:\ProgramData\WireGuard\vps-proxy.conf`。

## 6. 验收

```bash
# 服务器侧
wg show                          # peer 最近握手时间在刷新
ss -lunp | grep 52800            # 监听在位

# 客户端（经 mihomo mixed-port 7897，先在 OpenAI 组选中本线路节点）
curl -x http://127.0.0.1:7897 -s https://checkip.amazonaws.com        # = VPS 公网 IP
curl -x http://127.0.0.1:7897 -o /dev/null -w '%{http_code}\n' https://api.openai.com/v1/models   # 401 = 可达
curl -x http://127.0.0.1:7897 -o /dev/null -w '%{http_code}\n' https://chatgpt.com                # 403 = 可达（CF 防爬）
```

401/403 都是"网络可达"的正常信号——OpenAI 对未带凭据请求就是这个返回；000/超时才是故障。
