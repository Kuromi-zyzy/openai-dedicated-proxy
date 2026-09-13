# 02 · 容灾线路：Xray VLESS + REALITY (TCP 443)

> 目标：第二台 VPS（建议与主力**不同地域**，做线路对照/容灾）跑 Xray VLESS+REALITY Vision，TCP 443 以 `www.bing.com` 为伪装目标，公网直出。
> 执行 runbook 见 [AGENT_RUNBOOK.md](../AGENT_RUNBOOK.md) §5；本文讲原理、选型与防误诊。
> **版本纪律：锁定 v25.3.31，不自动升级。**

## 0. 防火墙要求

| 端口 | 协议 | 放行 | 用途 |
|------|------|------|------|
| 22 | TCP | 是（建议收敛源 IP） | SSH 管理 |
| 443 | TCP | 是 | REALITY 入口 |

机器内防火墙（ufw/firewalld）增量放行 443/tcp + 云控制台安全组放行 443/tcp，两层都要。

## 1. 安装（版本锁定）

**不要用官方一键脚本默认安装**——它总拉最新版，而 26.x 在本方案实测存在静默失败。直接下载 v25.3.31 release 包：

```bash
cd /tmp
curl -LO https://github.com/XTLS/Xray-core/releases/download/v25.3.31/Xray-linux-64.zip
apt install -y unzip && unzip Xray-linux-64.zip
install -m 755 xray /usr/local/bin/xray
xray version   # 必须显示 25.3.31
```

专用系统用户（**不要用默认的 nobody**）：

```bash
groupadd -r xray && useradd -r -g xray -s /usr/sbin/nologin -M xray
mkdir -p /var/log/xray && chown -R xray:xray /var/log/xray
```

## 2. 密钥与 UUID

```bash
xray x25519
# v25 输出：Private key: <服务端私钥> / Public key: <服务端公钥，给客户端 reality-opts.public-key>
```

> ⚠️ **v26 起 `xray x25519` 输出格式变了**（`PrivateKey:` / `Password (PublicKey):` / `Hash32:`）。脚本按旧字段名截取会拿到空值且不报错——生成后**逐字段人工核对**。

```bash
xray uuid            # 客户端 uuid
openssl rand -hex 8  # shortId
```

## 3. target 选型：用 bing，别用 microsoft（血泪坑）

REALITY 需要一个真实 TLS 站点做伪装目标（`target` / `serverNames`）。**实测 `www.microsoft.com` 不能用**：它带 OCSP stapling，证书记录 8273 字节，超过 Xray REALITY 代码硬编码的 8192 上限，导致**所有客户端认证握手全部失败**（服务端日志 `REALITY: processed invalid connection: handshake did not complete successfully`），而**伪装回落路径完全正常**（openssl 直连能拿到真 microsoft 证书）——极具迷惑性，看起来像"网络不通"。对应 issue：XTLS/Xray-core#6356；v25/v26 全版本 + 5 种 uTLS 指纹均复现。

**用 `www.bing.com` 实测正常**（204 探测 + 真实 Codex 调用全通）。

## 4. 配置文件

`/usr/local/etc/xray/config.json`（root:xray 640），完整带注释版见 [examples/xray-server.example.jsonc](../examples/xray-server.example.jsonc)：

```json
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
        "clients": [
          { "id": "<UUID>", "flow": "xtls-rprx-vision" }
        ],
        "decryption": "none"
      },
      "streamSettings": {
        "method": "raw",
        "security": "reality",
        "realitySettings": {
          "show": false,
          "target": "www.bing.com:443",
          "serverNames": ["www.bing.com"],
          "privateKey": "<服务端私钥>",
          "shortIds": ["<shortId>"]
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
```

routing rules 是**安全硬要求**：把云元数据服务（IMDS `169.254.169.254`——各公有云通用的链路本地地址，主机角色/临时凭据就从这读）和全部私网段导向 blackhole。任何拿到你代理凭据的客户端都不应能经由出口访问这些地址。

## 5. systemd

自定义单元（非 root 用户运行 + 最小权限），`/etc/systemd/system/xray.service`：

```ini
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
```

```bash
systemctl daemon-reload && systemctl enable --now xray
ss -tlnp | grep ':443 '      # xray 监听
```

## 6. 客户端节点（mihomo）

```yaml
- name: VPS-REALITY
  type: vless
  server: <VPS_IP>
  port: 443
  uuid: <UUID>
  tls: true
  flow: xtls-rprx-vision
  servername: www.bing.com
  client-fingerprint: chrome
  reality-opts:
    public-key: <服务端公钥>
    short-id: <shortId>
  network: tcp
  udp: false
```

## 7. 验收与"误诊防护"

```bash
# 经代理的出口 IP 应为该 VPS 公网 IP
curl -x http://127.0.0.1:7897 https://checkip.amazonaws.com
# OpenAI 可达信号位
curl -x http://127.0.0.1:7897 -o /dev/null -w '%{http_code}\n' https://api.openai.com/v1/models    # 401
# 服务器 access.log：chatgpt.com / api.openai.com → direct；
# 169.254.169.254 / 127.0.0.1 / 10.x / 192.168.x → block（ACL 实证）
tail -f /var/log/xray/access.log
```

**误诊防护**（遇到故障按此顺序想，别急着怀疑网络）：

1. 所有客户端握手 EOF + 服务端 `handshake did not complete successfully` → target 是不是 microsoft 系（OCSP 大证书）；
2. 服务端**零日志**且回落正常 → 新版回归（26.9.9 实测连 bing target 都静默失败）→ 回退 v25.3.31；
3. `exit 23 / permission denied` 循环崩溃 → `/var/log/xray` 属主被重置 → `chown -R xray:xray /var/log/xray`。

## 8. 升级纪律

- 生产锁 v25.3.31；升级 = 非生产时段换二进制 + 全链路回归（含真实 Codex 调用），失败立即回退；
- **每次重装/升级后必做**：`chown -R xray:xray /var/log/xray`（官方 install 脚本会把日志属主重置回 root，非 root 用户启动即 permission denied）。
