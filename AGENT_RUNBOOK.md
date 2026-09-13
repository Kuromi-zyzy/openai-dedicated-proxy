# AGENT_RUNBOOK — OpenAI 专用代理部署执行手册

> **读者：AI agent。** 人类用户不用读这份，请看 [README.md](README.md)。
> **使命**：用户会给你一个 SSH 别名（或 `user@IP`）和部署档位（A / B / C）。你 SSH 上服务器，把档位对应的线路部署好、验收通过，然后产出客户端配置和凭据清单交付给用户。**默认单机部署**：档位 B/C 的 Xray/Hysteria2 与 WireGuard 装在同一台（端口互不冲突）；仅当用户明确提供第二台 SSH 目标并要求分开部署时才走双机模式。
> **自包含**：执行本手册不需要阅读 docs/ 其他文档；docs/ 是原理与排坑的深度参考。

---

## §0 执行纪律（动手前先读完）

1. **顺序执行** §1 → §7。每个阶段末尾有【校验】，校验不过：停在当前阶段，按本阶段【回滚】清理干净，向用户报告失败点，**不得跳步或带病继续**。
2. **占位符一次性定值**（§2）：能生成的自己生成，必须用户拍板的合并成一条消息问完，不要挤牙膏。
3. **硬红线**（任何一条将被触碰 → 立即停止并说明）：
   - 任何代理/SOCKS 监听**绝不绑 `0.0.0.0`**，1080 等"仅隧道内网"端口**绝不加进防火墙放行**；
   - **绝不修改 OpenSSH/sshd 配置**、不删除已有 authorized_keys（防止自锁）；
   - **绝不整体关闭/清空防火墙**，只做增量放行；
   - **版本锁定**：Xray 一律 `v25.3.31`，Hysteria2 一律 `v2.12.2`，不装"最新版"；
   - WireGuard 端口**绝不用 51820**（国内链路定向阻断）；
   - Xray / Hysteria2 的出口侧**必须** block 云元数据 `169.254.169.254` 与全部私网段——未配置不得交付。
4. **冲突即停**：若服务器上已存在 xray / hysteria / wireguard / nginx 等 443 或相关端口占用服务，或系统不是 Ubuntu 22.04+/Debian 12 (x86_64)，停下来问用户，不要覆盖、不要强行部署。
5. **凭据管理**：所有生成的密钥/密码写入服务器配置文件后，在 §7 交付报告里完整列出一份供用户保存，并提醒：不要发群聊、不要截图。
6. **shell 纪律**：写配置文件一律用 heredoc；内容含 `$` 变量或 `%i` 模板变量的，heredoc 定界符必须带引号（`<<'EOF'`）防止 shell 展开。远程执行长命令用 `ssh <别名> 'bash -s' <<'REMOTE'` 形式。
7. **排障升级路径**：遇到本文未覆盖的故障，查 `docs/05-troubleshooting.md` 对应坑；仍解决不了，如实向用户报告，不要瞎试超过 3 次。

---

## §1 你会收到的输入

| 输入 | 形式 | 缺省处理 |
|------|------|---------|
| SSH 目标 | `~/.ssh/config` 别名（如 `proxy-jp`）或 `root@1.2.3.4` | 必填，没有就问 |
| 档位 | A（WireGuard）/ B（A+Xray REALITY）/ C（B+Hysteria2） | 必填；用户只说"都搭"= C |
| VPS 用途备注 | 线路地域（日本/新加坡/美国…） | 可选，仅影响命名 |

档位与服务器对应关系（**默认单机，一台跑全部协议**）：

- 档位 A：服务器（下称 `主机甲`）跑 WireGuard。
- 档位 B/C：Xray（+HY2）**默认与 WG 装在同一台**——三个协议端口互不冲突（UDP 52800 / TCP 443 / UDP 443），一台 1C/1G 足够。
- 双机模式（可选）：**仅当**用户明确给出第二个 SSH 目标并要求"分开部署"时，Xray（+HY2）才装第二台（下称 `主机乙`）。不要主动建议用户为此购买第二台机器；如用户问起，说明单机=协议层容灾完整、双机=加地域级容灾。

---

## §2 占位符定值表

**用户提供的**（§1 已含，另问是否改端口）：

| 占位符 | 默认值 | 说明 |
|--------|--------|------|
| `<VPS_IP>` | — | 服务器公网 IP（双机模式下 Xray/HY2 目标单独记 `<VPS2_IP>`，单机模式两者相同） |
| `<WG_PORT>` | `52800` | WireGuard 端口，**禁 51820**，用户无偏好就用默认 |
| `<XRAY_PORT>` | `443` | Xray TCP 端口 |
| `<HY2_PORT>` | `443` | Hysteria2 UDP 端口（可与 Xray 同号不同协议） |

**Agent 自行生成的**（生成命令如下，跑完立即记录到临时变量与你的工作笔记）：

```bash
# WireGuard 密钥（每台参与的主机各一对服务端密钥；每个客户端各一对）
wg genkey | tee /tmp/wg_server.key | wg pubkey > /tmp/wg_server.pub
wg genkey | tee /tmp/wg_client.key  | wg pubkey > /tmp/wg_client.pub
# <WG_SERVER_PRIVATE_KEY> <WG_SERVER_PUBLIC_KEY> <WG_CLIENT_PRIVATE_KEY> <WG_CLIENT_PUBLIC_KEY>

# Xray
xray x25519            # → <XRAY_PRIVATE_KEY> / <XRAY_PUBLIC_KEY>（v25.3.31 输出为 "Private key:"/"Public key:"，逐字段核对，勿抄错列）
xray uuid              # → <XRAY_UUID>
openssl rand -hex 8    # → <XRAY_SHORT_ID>

# Hysteria2 密码 与 SOCKS5 密码
openssl rand -base64 24   # → <HY2_PASSWORD>
openssl rand -base64 24   # → <SOCKS_PASSWORD>（用户名固定 wgproxy）
```

隧道网段固定用 `10.66.66.0/24`（RFC1918 私网）：服务器 `10.66.66.1`，客户端 `10.66.66.2`（mihomo 用户态）与 `10.66.66.3`（Windows 内核态，备用）。

---

## §3 SSH 侦察（每台主机，约 5 分钟）

```bash
ssh <别名> 'bash -s' <<'REMOTE'
echo "=== 系统/内核 ==="; cat /etc/os-release | head -2; uname -m
echo "=== 公网出口 IP ==="; curl -s --max-time 8 https://checkip.amazonaws.com
echo "=== 端口占用（443/1080/52800 应无输出）==="
ss -tulnp | grep -E ':(443|1080|52800)\s' || echo "clean"
echo "=== TUN 设备（档位 A 需要）==="; ls -l /dev/net/tun 2>/dev/null || echo "NO TUN"
echo "=== 已装服务探测 ==="
systemctl is-active xray hysteria-server wg-quick@wg0 2>/dev/null
command -v xray hysteria wg microsocks 2>/dev/null || true
echo "=== 防火墙现状 ==="
command -v ufw >/dev/null && sudo ufw status verbose
command -v firewall-cmd >/dev/null && sudo firewall-cmd --state
echo "=== 时间同步（TLS 对时敏感）==="; timedatectl | grep -E 'synchronized|Time zone'
echo "=== 虚拟化/权限 ==="; sudo -n true && echo "sudo ok"; id
REMOTE
```

【校验】① 系统为 Ubuntu 22.04+/Debian 12 且 `uname -m` = x86_64；② 三个目标端口空闲；③ 档位 A 时 TUN 存在（OpenVZ/LXC 老容器可能没有 → 停，改建议档位 B）；④ 无同名服务在跑；⑤ `checkip` 返回值 = 用户说的 IP（不一致 → 问用户）。
【此阶段零改动，无需回滚】

---

## §4 档位 A：WireGuard + SOCKS5（主机甲 = 你的服务器）

以下命令均在主机甲执行（`ssh <别名>` 逐条或 `bash -s` 批量）。

### 4.1 安装与转发

```bash
apt update && apt install -y wireguard microsocks curl
echo 'net.ipv4.ip_forward = 1' > /etc/sysctl.d/99-wg-forward.conf
sysctl --system
```

### 4.2 密钥

按 §2 生成 `<WG_SERVER_*>` 与 `<WG_CLIENT_*>`（在服务器上生成，客户端私钥稍后随交付报告带给用户）。

### 4.3 查清出口网卡与主 IP（SNAT 用）

```bash
ip -4 route show default          # 默认路由网卡，通常是 eth0 → <NIC>
ip -4 addr show <NIC>             # 记下【挂公网的那个内网 IP】→ <MAIN_PRIVATE_IP>
```

> ⚠️ 若网卡上有多个内网 IP（部分云平台会挂辅助 IP 且辅助 IP 没绑公网），SNAT 必须**写死主 IP**；用 MASQUERADE 可能选错源地址，症状是"握手正常、客户端只发不收"。

### 4.4 写配置

`/etc/wireguard/wg0.conf`（完整参考 [examples/wg0.conf.example](examples/wg0.conf.example)）：

```bash
install -m 600 /dev/null /etc/wireguard/wg0.conf
cat > /etc/wireguard/wg0.conf <<'EOF'
[Interface]
Address = 10.66.66.1/24
MTU = 1380
ListenPort = 52800
PrivateKey = <WG_SERVER_PRIVATE_KEY>

# ① 转发放行（wg0 ↔ 出口网卡）
PostUp = iptables -I FORWARD 1 -i %i -o <NIC> -s 10.66.66.0/24 -j ACCEPT
PostUp = iptables -I FORWARD 1 -i <NIC> -o %i -d 10.66.66.0/24 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
# ② SNAT 写死主内网 IP（勿用 MASQUERADE，原因见 4.3）
PostUp = iptables -t nat -I POSTROUTING 1 -s 10.66.66.0/24 -o <NIC> -j SNAT --to-source <MAIN_PRIVATE_IP>
# ③ MSS clamp：修复 OpenAI(Cloudflare) 回包 mss 1400 > 隧道 MTU 1380 的分片丢包
PostUp = iptables -t mangle -I FORWARD 1 -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu

PostDown = iptables -D FORWARD -i %i -o <NIC> -s 10.66.66.0/24 -j ACCEPT
PostDown = iptables -D FORWARD -i <NIC> -o %i -d 10.66.66.0/24 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
PostDown = iptables -t nat -D POSTROUTING -s 10.66.66.0/24 -o <NIC> -j SNAT --to-source <MAIN_PRIVATE_IP>
PostDown = iptables -t mangle -D FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu

[Peer]
PublicKey = <WG_CLIENT_PUBLIC_KEY>
AllowedIPs = 10.66.66.2/32

[Peer]
PublicKey = <WG_CLIENT3_PUBLIC_KEY>
AllowedIPs = 10.66.66.3/32
EOF
```

> 写完**逐行核对**三个 `<NIC>`、两个 `<MAIN_PRIVATE_IP>`、两处密钥都已替换；`%i` 是 wg-quick 模板变量，保持原样。

### 4.5 启动

```bash
systemctl enable --now wg-quick@wg0
```

### 4.6 防火墙增量放行（三分支，按机器实际有的来）

```bash
# Ubuntu ufw：
ufw allow 52800/udp comment 'WireGuard (never use 51820)'
# firewalld：
firewall-cmd --permanent --add-port=52800/udp && firewall-cmd --reload
# 两者都没有则不动 iptables（PostUp 已放行转发）；提醒用户去云控制台安全组放行 UDP 52800
```

同时**提醒用户**在云控制台/安全组放行 `UDP <WG_PORT>`（这是防火墙之外的另一层，agent 通常够不着）。

### 4.7 microsocks（SOCKS5，只听隧道内网）

```bash
cat > /etc/microsocks-wg.env <<'EOF'
MSPASS=<SOCKS_PASSWORD>
EOF
chmod 600 /etc/microsocks-wg.env

cat > /etc/systemd/system/microsocks-wg.service <<'EOF'
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
EOF
systemctl daemon-reload && systemctl enable --now microsocks-wg
```

【校验·档位 A】

```bash
wg show                      # interface wg0，监听 <WG_PORT>
ss -lunp | grep 52800        # udp 监听在位
ss -tlnp | grep 1080         # 必须显示 10.66.66.1:1080 —— 出现 0.0.0.0:1080 属事故，立即回滚 4.7
sysctl net.ipv4.ip_forward   # = 1
```

【回滚·档位 A】

```bash
systemctl disable --now microsocks-wg wg-quick@wg0
rm -f /etc/systemd/system/microsocks-wg.service /etc/microsocks-wg.env /etc/wireguard/wg0.conf /etc/sysctl.d/99-wg-forward.conf
systemctl daemon-reload
# 防火墙：删除本次加的 52800/udp 规则（ufw delete allow 52800/udp）
```

---

## §5 档位 B：Xray REALITY（TCP 443；默认与 WG 同机，双机模式=主机乙）

### 5.1 版本锁定安装

```bash
cd /tmp
curl -LO https://github.com/XTLS/Xray-core/releases/download/v25.3.31/Xray-linux-64.zip
apt install -y unzip && unzip -o Xray-linux-64.zip
install -m 755 xray /usr/local/bin/xray
xray version    # 必须显示 25.3.31，否则停下
```

专用用户与日志目录：

```bash
groupadd -r xray 2>/dev/null; useradd -r -g xray -s /usr/sbin/nologin -M xray 2>/dev/null
mkdir -p /var/log/xray && chown -R xray:xray /var/log/xray
```

### 5.2 密钥

按 §2 生成 `<XRAY_PRIVATE_KEY>` `<XRAY_PUBLIC_KEY>` `<XRAY_UUID>` `<XRAY_SHORT_ID>`。**生成后逐字段人工核对**（不同版本 x25519 输出列名不同，抄错列 = 全灭且不报错）。

### 5.3 配置（target 必须 bing，禁用 microsoft）

`/usr/local/etc/xray/config.json`（完整参考 [examples/xray-server.example.jsonc](examples/xray-server.example.jsonc)）：

```bash
install -m 640 -o root -g xray /dev/null /usr/local/etc/xray/config.json
cat > /usr/local/etc/xray/config.json <<'EOF'
{
  "log": {
    "access": "/var/log/xray/access.log",
    "error": "/var/log/xray/error.log",
    "loglevel": "warning"
  },
  "inbounds": [
    {
      "tag": "reality-vision",
      "listen": "::",
      "port": 443,
      "protocol": "vless",
      "settings": {
        "clients": [ { "id": "<XRAY_UUID>", "flow": "xtls-rprx-vision" } ],
        "decryption": "none"
      },
      "streamSettings": {
        "method": "raw",
        "security": "reality",
        "realitySettings": {
          "show": false,
          "target": "www.bing.com:443",
          "serverNames": ["www.bing.com"],
          "privateKey": "<XRAY_PRIVATE_KEY>",
          "shortIds": ["<XRAY_SHORT_ID>"]
        }
      },
      "sniffing": { "enabled": true, "destOverride": ["http", "tls", "quic"], "metadataOnly": false }
    }
  ],
  "outbounds": [
    { "tag": "direct", "protocol": "freedom", "settings": {} },
    { "tag": "block", "protocol": "blackhole", "settings": {} }
  ],
  "routing": {
    "domainStrategy": "IPOnDemand",
    "rules": [
      {
        "type": "field",
        "ip": [
          "169.254.169.254/32",
          "0.0.0.0/8", "10.0.0.0/8", "100.64.0.0/10", "127.0.0.0/8",
          "169.254.0.0/16", "172.16.0.0/12", "192.168.0.0/16",
          "::1/128", "fc00::/7", "fe80::/10"
        ],
        "outboundTag": "block"
      }
    ]
  }
}
EOF
```

> routing rules 是**硬要求**：block 云元数据（IMDS，云角色临时凭据在这）+ 全部私网段。

### 5.4 systemd

```bash
cat > /etc/systemd/system/xray.service <<'EOF'
[Unit]
Description=Xray Service
After=network.target nss-lookup.target

[Service]
User=xray
Group=xray
ExecStart=/usr/local/bin/xray run -config /usr/local/etc/xray/config.json
Restart=on-failure
RestartSec=5
NoNewPrivileges=true

[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload && systemctl enable --now xray
```

### 5.5 防火墙与安全组

增量放行 `TCP 443`（ufw: `ufw allow 443/tcp`；firewalld: `--add-port=443/tcp`）+ 提醒用户云安全组放行 TCP 443。

【校验·档位 B】

```bash
xray run -test -config /usr/local/etc/xray/config.json   # 先于启动执行，配置合法性
systemctl is-active xray                                  # active
ss -tlnp | grep ':443 '                                   # xray 监听
# 回落伪装正常（能拿到 bing 真证书说明 target 可达）：
openssl s_client -connect 127.0.0.1:443 -servername www.bing.com </dev/null 2>/dev/null | openssl x509 -noout -issuer
```

【回滚·档位 B】

```bash
systemctl disable --now xray
rm -f /etc/systemd/system/xray.service /usr/local/etc/xray/config.json
rm -f /usr/local/bin/xray && userdel xray 2>/dev/null; groupdel xray 2>/dev/null
rm -rf /var/log/xray
```

> 重装/重配后必做：`chown -R xray:xray /var/log/xray`——否则非 root 用户启动即 `permission denied` 循环崩溃。

---

## §6 档位 C：Hysteria2（UDP 443，与 Xray 同机共存）

### 6.1 版本锁定安装 + 校验

```bash
cd /tmp
curl -LO https://github.com/apernet/hysteria/releases/download/app%2Fv2.12.2/hysteria-linux-amd64
curl -LO https://github.com/apernet/hysteria/releases/download/app%2Fv2.12.2/hashes.txt
# hashes.txt 的文件名带 "build/" 前缀，按此匹配：
awk '$2=="build/hysteria-linux-amd64"' hashes.txt    # 记下期望 SHA-256
sha256sum hysteria-linux-amd64                        # 人工比对一致后继续
install -m 755 hysteria-linux-amd64 /usr/local/bin/hysteria
hysteria version | head -3
```

### 6.2 专用用户 + 自签证书（锁指纹，不用 skip-cert-verify）

```bash
groupadd -r hysteria 2>/dev/null; useradd -r -g hysteria -s /usr/sbin/nologin -M hysteria 2>/dev/null
mkdir -p /etc/hysteria && chown root:hysteria /etc/hysteria && chmod 750 /etc/hysteria

# ⚠️ 目录 750 会让普通用户 shell 无法展开 *.pem glob —— 下面全部用显式路径，绝不写通配符
openssl ecparam -genkey -name prime256v1 -out /etc/hysteria/key.pem
openssl req -new -x509 -days 3650 -key /etc/hysteria/key.pem -out /etc/hysteria/cert.pem -subj "/CN=www.bing.com"
chown root:hysteria /etc/hysteria/cert.pem /etc/hysteria/key.pem
chmod 640 /etc/hysteria/cert.pem /etc/hysteria/key.pem

# 证书指纹（交付给客户端锁定，防中间人）：
openssl x509 -in /etc/hysteria/cert.pem -noout -fingerprint -sha256
# 输出形如 sha256 Fingerprint=AA:BB:… → 去掉前缀与冒号后的小写十六进制 = <HY2_CERT_SHA256>
```

### 6.3 配置（ACL 是硬要求）

`/etc/hysteria/config.yaml`（完整参考 [examples/hysteria2-server.example.yaml](examples/hysteria2-server.example.yaml)）：

```bash
cat > /etc/hysteria/config.yaml <<'EOF'
listen: :443

tls:
  cert: /etc/hysteria/cert.pem
  key: /etc/hysteria/key.pem

auth:
  type: password
  password: <HY2_PASSWORD>

disableUDP: true

acl:
  inline:
    - reject(169.254.0.0/16)
    - reject(0.0.0.0/8)
    - reject(10.0.0.0/8)
    - reject(100.64.0.0/10)
    - reject(127.0.0.0/8)
    - reject(172.16.0.0/12)
    - reject(192.168.0.0/16)
    - reject(198.18.0.0/15)
    - reject(224.0.0.0/4)
    - reject(::1/128)
    - reject(fc00::/7)
    - reject(fe80::/10)
EOF
chown root:hysteria /etc/hysteria/config.yaml && chmod 640 /etc/hysteria/config.yaml
```

> ACL 语法注意：`reject(cidr)` **省略端口 = 拒绝该网段全部端口**；`all` 不是合法网段写法。这组 reject 是硬要求（同 Xray：IMDS + 私网全拒）。

### 6.4 systemd（硬化）

```bash
cat > /etc/systemd/system/hysteria-server.service <<'EOF'
[Unit]
Description=Hysteria2 Server
After=network-online.target
Wants=network-online.target

[Service]
User=hysteria
Group=hysteria
ExecStart=/usr/local/bin/hysteria server -c /etc/hysteria/config.yaml
Restart=always
RestartSec=5
NoNewPrivileges=true
ProtectHome=true
ProtectSystem=strict
PrivateTmp=true
AmbientCapabilities=CAP_NET_BIND_SERVICE
CapabilityBoundingSet=CAP_NET_BIND_SERVICE

[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload && systemctl enable --now hysteria-server
```

### 6.5 防火墙与安全组

增量放行 `UDP 443`（ufw: `ufw allow 443/udp`）+ 提醒用户云安全组放行 UDP 443。

【校验·档位 C】

```bash
systemctl is-active hysteria-server       # active，且 restart 计数为 0（若在循环重启 → 见下方坑）
ss -ulnp | grep ':443 '                   # hysteria 监听 UDP 443
journalctl -u hysteria-server -n 20 --no-pager   # 无 permission denied / config error
```

【回滚·档位 C】

```bash
systemctl disable --now hysteria-server
rm -f /etc/systemd/system/hysteria-server.service
rm -rf /etc/hysteria && rm -f /usr/local/bin/hysteria
userdel hysteria 2>/dev/null; groupdel hysteria 2>/dev/null
```

> **两个高发坑**：① 服务循环重启且日志 `permission denied` → 检查 cert/key/config 三件属主必须 `root:hysteria` 640、目录 750（见 6.2 的显式路径 chown 原因）；② ACL 写错（如用了 `all`）→ 服务起不来或拒绝规则不生效，按 6.3 原样抄。

---

## §7 产出客户端配置、验收、交付

### 7.1 生成 mihomo（Clash Verge）节点片段

按实际部署拼装（模板见 [examples/clash/nodes-merge.example.yaml](examples/clash/nodes-merge.example.yaml) 与 [examples/clash/groups-rules.example.yaml](examples/clash/groups-rules.example.yaml)）：

- 档位 A → `VPS-WG`（mihomo wireguard 用户态直连，**必给**：不依赖 Windows 内核客户端，开箱即用）与 `VPS-WG-SOCKS`（socks5，仅在 Windows 内核态 WG 就绪后给）：
  ```yaml
  - name: VPS-WG
    type: wireguard
    server: <VPS_IP>
    port: 52800
    ip: 10.66.66.2
    private-key: <WG_CLIENT_PRIVATE_KEY>
    public-key: <WG_SERVER_PUBLIC_KEY>
    udp: true
    mtu: 1380
    persistent-keepalive: 25
    remote-dns-resolve: true
    dns: [1.1.1.1]
  ```
- 档位 B → `VPS-REALITY`（vless 节点，字段见 5.3 配置对应值）
- 档位 C → `VPS-HY2`（hysteria2 节点：`password/sni www.bing.com/fingerprint <HY2_CERT_SHA256>/up 30/down 100/udp false`）
- 策略组：`select` 手动组，成员按 A/B/C 顺序排列，**禁止** url-test/load-balance/fallback
- 规则（放 rules **最顶部**）：`DOMAIN-SUFFIX,openai.com / chatgpt.com / oaistatic.com / oaiusercontent.com / oaistudio.com` 五条 → 组名

### 7.2 引导用户粘贴（人类 GUI 操作）

1. Clash Verge → 配置/Profiles → 当前订阅右键「编辑 Merge/增强」（或新建增强配置），粘贴节点与组片段；
2. 规则片段粘进 Merge 的 `rules` 前置（prepend）；
3. 保存并**重启内核**（Verge 界面「重启内核」，不要只热重载——对 wireguard 节点，API 热重载可能把出站弄挂）；
4. 「OpenAI」组里选中主节点。

### 7.3 验收矩阵（在用户本机 / agent 本地 shell 执行）

```bash
# 经 mihomo mixed-port（默认 7897）逐线路测：
curl -x http://127.0.0.1:7897 -s https://checkip.amazonaws.com        # = 对应 VPS 公网 IP（出口固定性）
curl -x http://127.0.0.1:7897 -s -o /dev/null -w '%{http_code}\n' https://api.openai.com/v1/models   # 401 = 可达信号位
curl -x http://127.0.0.1:7897 -s -o /dev/null -w '%{http_code}\n' https://chatgpt.com                # 403 = 可达信号位（CF 防爬）
```

- 401/403 都是**成功信号**（未带凭据的应有返回）；000/超时才是失败。
- 切到哪条线路，checkip 就应返回哪台的 IP——同组多线路出口必须各自固定。

### 7.4 交付报告（发给用户的最终消息，按此模板）

```
✅ 部署完成：<档位> · 主机：<别名 / IP / 线路地域>

【客户端配置】（粘贴方法见 7.2）
<nodes + groups + rules 完整 YAML 片段>

【凭据清单——请保存到本地安全位置，勿发群聊/截图】
- WG 客户端私钥 / 服务器公钥：<值>（文件位置 /etc/wireguard/wg0.conf）
- SOCKS5：wgproxy / <SOCKS_PASSWORD>（仅隧道内可用，永不公网）
- Xray UUID / publicKey / shortId：<值>（/usr/local/etc/xray/config.json）
- HY2 密码 / 证书指纹：<值>（/etc/hysteria/config.yaml）

【验收结果】
- checkip：…；api.openai.com：401；chatgpt.com：403

【云控制台待办】（若 agent 无权限）
- 安全组放行：UDP 52800（档位 A）/ TCP 443 + UDP 443（档位 B/C）

【回滚命令】<各已部署档位的回滚块>

【注意事项】
- 版本已锁定 v25.3.31 / v2.12.2，勿自动升级
- 若 WireGuard 突然"能发不能收"：先按 docs/05 坑 1 换端口，别怀疑实现
- 面板延迟 400~500ms 属正常（多 RTT 叠加）
```

---

## 附录 A · 故障速查（详版 → docs/05-troubleshooting.md）

| 症状 | 先查 | 处置 |
|------|------|------|
| WG 握手正常但客户端只发不收 | SNAT 是否写死主 IP（4.3） | MASQUERADE → SNAT |
| WG 昨天还好今天全断，TCP 一切正常 | 是否用了 51820 或端口被盯 | UDP 回显判别（坑 1）→ 换端口 |
| 连接间歇性 dial 超时/SACK 重传 | MSS clamp 是否在位（4.4 ③） | 补 TCPMSS 规则 |
| Xray 所有握手 EOF，服务端 `handshake did not complete successfully` | target 是否 microsoft | 换 www.bing.com（坑 5） |
| xray exit 23 / permission denied | /var/log/xray 属主 | chown -R xray:xray |
| HY2 循环重启 | cert/key/config 属主与目录权限 | 按 6.2 显式路径修 |
| 热重载后 WG 节点 Timeout | mihomo API 热重载副作用 | 界面重启内核 |

## 附录 B · 原理与深度参考

- [docs/01-wireguard-socks.md](docs/01-wireguard-socks.md) · [docs/02-xray-reality.md](docs/02-xray-reality.md) · [docs/03-hysteria2.md](docs/03-hysteria2.md) · [docs/04-clash-verge-client.md](docs/04-clash-verge-client.md)
