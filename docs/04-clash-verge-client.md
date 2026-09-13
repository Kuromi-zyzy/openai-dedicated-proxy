# 04 · 客户端：Clash Verge (mihomo) 配置

> 目标：Windows 上用 Clash Verge 承载全部线路，OpenAI 生态流量进专用策略组，其余流量不经过本方案。
> 执行时 Agent 会直接产出下文各片段的成品值；本文讲写法、机制与坑。

## 1. 总体结构

Clash Verge 的推荐组织方式（订阅可继续用你现有的，本方案全部通过 **Merge/增强配置** 注入，不污染原始订阅）：

```
订阅 profile（机场/自建，提供默认规则与出口）
   + Merge 增强 ①：proxies 追加本方案节点        ← examples/clash/nodes-merge.example.yaml
   + Merge 增强 ②：proxy-groups 追加 OpenAI 组
   + 规则前置（prepend）：OpenAI 域名 → OpenAI 组  ← examples/clash/groups-rules.example.yaml
```

分流规则五条，**放在 rules 最顶部**（优先级最高，先于订阅自带规则）：

```yaml
rules:
  - DOMAIN-SUFFIX,openai.com,OpenAI
  - DOMAIN-SUFFIX,chatgpt.com,OpenAI
  - DOMAIN-SUFFIX,oaistatic.com,OpenAI      # 静态资源
  - DOMAIN-SUFFIX,oaiusercontent.com,OpenAI # 用户内容
  - DOMAIN-SUFFIX,oaistudio.com,OpenAI
```

> Codex CLI / ChatGPT 桌面端 / API 的域名都被这五条覆盖（`codex.openai.com`、`api.openai.com` 等是上述后缀的子域）。

## 2. 策略组：手动 select，绝不让它自动漂移

```yaml
proxy-groups:
  - name: OpenAI
    type: select            # 手动！
    proxies:
      - VPS-WG-SOCKS        # 内核态 WG → 服务器 SOCKS5（若部署了接法 B）
      - VPS-WG              # mihomo 用户态 WG
      - VPS-REALITY         # VLESS+REALITY（档位 B）
      - VPS-HY2             # Hysteria2（档位 C）
```

**为什么禁止 `url-test` / `load-balance` / `fallback`**：Codex agent 任务是十几分钟级的 TCP 长连接，组内自动切节点 = 掐断会话；且出口 IP 漂移会触发 OpenAI 风控。`select` 保证出口只在人点的时候变。

**混合部署时的成员顺序**：把当前最稳的放第一位（默认选中）。

## 3. 四种节点写法速查

完整字段说明见各线路文档，这里只列易错点：

**socks5（内核态 WG 的出口）**——`server` 填 **WG 隧道内的服务器地址**（`10.66.66.1`），不是公网 IP；`udp: false`（microsocks 无 UDP ASSOCIATE，无影响）：

```yaml
- name: VPS-WG-SOCKS
  type: socks5
  server: 10.66.66.1
  port: 1080
  username: wgproxy
  password: <SOCKS_PASSWORD>
  udp: false
```

**wireguard（mihomo 用户态）**——`ip` 是客户端在隧道内的地址（对应服务器 peer 的 AllowedIPs）；`public-key` 是**服务器**公钥：

```yaml
- name: VPS-WG
  type: wireguard
  server: <VPS_IP>
  port: 52800
  ip: 10.66.66.2
  private-key: <客户端私钥>
  public-key: <服务器公钥>
  udp: true
  mtu: 1380
  persistent-keepalive: 25
  remote-dns-resolve: true
  dns: [1.1.1.1]
```

**vless + REALITY**——`servername` 与服务端 `serverNames` 一致；`client-fingerprint` 用 `chrome`：

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

**hysteria2**——`fingerprint` 填服务器自签证书的 SHA-256 指纹（锁指纹，勿用 `skip-cert-verify: true`）；`up/down` 按家宽实际填，宁可小：

```yaml
- name: VPS-HY2
  type: hysteria2
  server: <VPS_IP>
  port: 443
  password: <HY2_PASSWORD>
  sni: www.bing.com
  fingerprint: <证书SHA256指纹>
  up: 30
  down: 100
  udp: false
```

## 4. 粘贴位置（GUI 操作）

1. Clash Verge → 「订阅」页 → 当前 profile 右键 → **编辑 Merge（增强配置）**；
2. 节点片段放进 `append.proxies`，组片段放进 `append.proxy-groups`（格式见 examples）；
3. 规则五条放进 Merge 的 `prepend-rules`（Verge 会插到 rules 最前）；
4. 保存后**重启内核**（左侧「设置」→ 重启内核，或直接重启 Verge）；
5. 「代理」页 → OpenAI 组 → 点选主节点。

## 5. 热重载的坑（重要）

- **对 wireguard 节点，mihomo API 热重载（`PUT /configs?force=true`）可能把 WG 出站弄挂**（实测：重载后 dial 超时、节点 Timeout，`POST /restart` 重建内核才恢复）。日常改配置**用界面「重启内核」**，不用纯热重载。
- **只改增强文件、运行时文件没更新时，重载不生效**——Verge 的热重载读的是合并后的运行时配置；改完增强配置后在界面保存/触发一次完整重载，确认面板里新节点出现。

## 6. 验证

```bash
# 假设 mihomo mixed-port = 7897（Verge 默认）
# ① 出口固定性：切到哪条线路，就应返回哪台 VPS 的 IP
curl -x http://127.0.0.1:7897 -s https://checkip.amazonaws.com
# ② 可达信号位：401/403 都=网络通；000/超时=故障
curl -x http://127.0.0.1:7897 -o /dev/null -w '%{http_code}\n' https://api.openai.com/v1/models   # 401
curl -x http://127.0.0.1:7897 -o /dev/null -w '%{http_code}\n' https://chatgpt.com                # 403
# ③ 出口落地 PoP（可选）：看 CF-RAY 结尾的地域码
curl -x http://127.0.0.1:7897 -sI https://chatgpt.com | grep -i cf-ray
```

**真实 Codex 调用验收**（可选，最终级）：

```bash
# 让 Codex CLI 走本机 7897（即走 OpenAI 组当前选中节点）
HTTPS_PROXY=http://127.0.0.1:7897 codex exec "Reply with exactly: OK"
```

## 7. Codex CLI / 其他工具怎么接到这个代理

- **Clash 系统代理模式**（Verge 默认）：浏览器走规则分流，OpenAI 域名自动进组；
- **命令行工具**（Codex CLI 等）：显式注入 `HTTPS_PROXY=http://127.0.0.1:7897`（或 `HTTP_PROXY`/`ALL_PROXY`）；
- **TUN 模式**：可选开启，让不认代理的软件也走分流；注意 TUN 会接管全局，确认规则表里大流量域名（Steam/更新）不在 OpenAI 组。

## 8. 面板延迟的正确解读

策略组节点上显示的 400~500ms **是正常的**：测速 = HTTP 请求经隧道的 3~5 个 RTT 叠加（例如移动→日本路由 ~128ms/RTT）。商业机场 50ms 是 1~2 个优化路由 RTT，没有可比性。**判定故障看 7.3 的信号位与真实调用，不看面板延迟数字。**
