# 03 · 第三保险：Hysteria2 (QUIC/UDP 443)

> 目标：与 Xray **同机**部署 Hysteria2，监听 UDP 443（与 Xray 的 TCP 443 同号不同协议，互不冲突）。QUIC/HTTP-3 伪装形态，高丢包链路上的兜底线路。
> 执行 runbook 见 [AGENT_RUNBOOK.md](../AGENT_RUNBOOK.md) §6。
> **版本纪律：锁定 v2.12.2，永不自动升级。**

## 0. 定位

- 前向纠错 + 抗丢包：链路劣化窗口（丢包 10~20%）下体验最好，作为 REALITY/WG 都难受时的兜底；
- UDP 443 与 TCP 443 共存：安全组/防火墙各放行一条即可，端口不冲突；
- 弱点：部分运营商对 UDP 有 QoS，所以它是"第三保险"而非主力。

防火墙：增量放行 **UDP 443**（机器内 + 云控制台两层）。

## 1. 安装（版本锁定 + 校验和）

```bash
cd /tmp
curl -LO https://github.com/apernet/hysteria/releases/download/app%2Fv2.12.2/hysteria-linux-amd64
curl -LO https://github.com/apernet/hysteria/releases/download/app%2Fv2.12.2/hashes.txt
```

> ⚠️ hashes.txt 里的文件名带 `build/` 前缀（`build/hysteria-linux-amd64`），awk 匹配时别写错：

```bash
awk '$2=="build/hysteria-linux-amd64"' hashes.txt   # 期望 SHA-256
sha256sum hysteria-linux-amd64                       # 人工比对一致再装
install -m 755 hysteria-linux-amd64 /usr/local/bin/hysteria
hysteria version
```

专用系统用户：

```bash
groupadd -r hysteria && useradd -r -g hysteria -s /usr/sbin/nologin -M hysteria
```

## 2. 自签证书（锁指纹，不用 skip-cert-verify）

```bash
mkdir -p /etc/hysteria && chown root:hysteria /etc/hysteria && chmod 750 /etc/hysteria

# ⚠️ 目录 750 时普通用户 shell 无法展开 *.pem 通配符 → chown 静默失败 → 服务循环重启。
# 全部用显式路径，绝不写通配符：
openssl ecparam -genkey -name prime256v1 -out /etc/hysteria/key.pem
openssl req -new -x509 -days 3650 -key /etc/hysteria/key.pem -out /etc/hysteria/cert.pem -subj "/CN=www.bing.com"
chown root:hysteria /etc/hysteria/cert.pem /etc/hysteria/key.pem
chmod 640 /etc/hysteria/cert.pem /etc/hysteria/key.pem
```

生成客户端要用的指纹（pinSHA256，防中间人）：

```bash
openssl x509 -in /etc/hysteria/cert.pem -noout -fingerprint -sha256
# 输出 sha256 Fingerprint=AA:BB:CC:… → 去掉前缀与冒号的小写十六进制，填进客户端 fingerprint 字段
```

自签 ECC P-256 证书 + 客户端锁定指纹 = 免去买域名/证书，又不牺牲防中间人（**不要**用 `skip-cert-verify: true`）。

## 3. 配置（ACL 是硬要求）

`/etc/hysteria/config.yaml`（root:hysteria 640），完整版见 [examples/hysteria2-server.example.yaml](../examples/hysteria2-server.example.yaml)：

```yaml
listen: :443

tls:
  cert: /etc/hysteria/cert.pem
  key: /etc/hysteria/key.pem

auth:
  type: password
  password: <强随机密码>

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
```

要点：

- **ACL 必须配**：Hysteria2 默认对目标地址无过滤，不配 ACL 等于任何人拿到密码就能经你的服务器访问内网/云元数据。实测配好后 IMDS/loopback/10.x 全部 DENIED、公网正常放行。
- **ACL 语法坑**：`reject(cidr)` 省略端口 = 拒绝该网段**全部端口**；`all` 不是合法写法（服务端不认）。
- `disableUDP: true`：服务端关闭 UDP 中继（OpenAI 生态是 TCP 长连接，用不到，缩小攻击面）。

## 4. systemd（硬化）

`/etc/systemd/system/hysteria-server.service`，完整版见 [examples/hysteria2-server.service.example](../examples/hysteria2-server.service.example)：

```ini
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
```

```bash
systemctl daemon-reload && systemctl enable --now hysteria-server
systemctl reset-failed hysteria-server 2>/dev/null   # 清历史失败计数，取干净基线
ss -ulnp | grep ':443 '    # UDP 443 监听
```

## 5. 服务器本地 ACL 实测（交付前必做）

在服务器本机起一个临时 hysteria 客户端（socks5 只听 127.0.0.1 高位端口），走完整 HY2 链路验证 ACL：

```bash
cat > /tmp/hy2-test.yaml <<'EOF'
server: 127.0.0.1:443
auth: <HY2_PASSWORD>
tls:
  sni: www.bing.com
  insecure: false
  cert: /etc/hysteria/cert.pem
socks5:
  listen: 127.0.0.1:11080
EOF
hysteria client -c /tmp/hy2-test.yaml &

sleep 3
curl -x socks5h://127.0.0.1:11080 --max-time 8 http://169.254.169.254/ ; echo "IMDS → 应被拒（exit 97/超时）"
curl -x socks5h://127.0.0.1:11080 --max-time 8 http://127.0.0.1:22/ ; echo "loopback → 应被拒"
curl -x socks5h://127.0.0.1:11080 -s https://checkip.amazonaws.com ; echo "公网 → 应返回本机公网 IP"

kill %1 && rm -f /tmp/hy2-test.yaml
```

## 6. 客户端节点（mihomo）

```yaml
- name: VPS-HY2
  type: hysteria2
  server: <VPS_IP>
  port: 443
  password: <HY2_PASSWORD>
  sni: www.bing.com
  fingerprint: <证书 SHA-256 指纹，十六进制>
  up: 30          # 上行 Mbps，按实际家宽填，宁可小
  down: 100       # 下行 Mbps
  udp: false
```

## 7. 验收

```bash
# 切到该节点后（客户端）：
curl -x http://127.0.0.1:7897 -s https://checkip.amazonaws.com     # = VPS 公网 IP（与 REALITY 同机则相同）
curl -x http://127.0.0.1:7897 -o /dev/null -w '%{http_code}\n' https://chatgpt.com   # 403 = 可达
# 服务器日志：家宽 IP 的 QUIC 认证成功记录
journalctl -u hysteria-server -n 20 --no-pager
```

## 8. 高发坑速记（详版见排坑手册）

1. **循环重启 + permission denied**：cert/key/config 属主没对（须 `root:hysteria` 640、目录 750）——通配符展开失败导致 chown 静默漏掉文件是根因；
2. **ACL 写 `all`**：服务端不认，规则不生效或起不来；
3. **hashes.txt 匹配不上**：文件名带 `build/` 前缀。
