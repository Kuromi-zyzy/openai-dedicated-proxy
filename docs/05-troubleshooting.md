# 05 · 排坑手册（13 个实战坑：症状 → 判别 → 根因 → 处置）

> 全部来自生产环境真实踩坑。**先读这篇再部署，能省一天的误诊时间**；部署后遇到故障按编号速查。
> 标 ✦ 的是最容易误诊、代价最大的坑。

---

## ✦ 坑 1：UDP 51820（WireGuard 默认端口）被国内链路定向阻断

- **症状**：隧道"突然全断"且持续数小时——客户端握手请求能到服务器、服务器 0.3ms 内正常回包，**但客户端永远收不到回应**；同期 TCP/SSH 一切正常、其它国外 UDP 节点正常。极易误判成"软件不稳定"（我们曾误判过，还为此错误归因了客户端实现）。
- **判别方法**（可复用，15 分钟定案）：在服务器**目标端口**跑一个普通 UDP 回显探针：

  ```bash
  # 服务器上：目标端口跑 UDP 回显（需临时在防火墙/安全组放行该端口）
  python3 - <<'EOF'
  import socket
  s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
  s.bind(("0.0.0.0", 51820))
  while True:
      data, addr = s.recvfrom(2048)
      print("recv from", addr)
      s.sendto(b"PONG:" + data, addr)
  EOF
  ```
  从客户端向该端口发包：**服务器日志显示收到、客户端收不到回显 = 该方向被链路丢**；换个端口（如 52800）同法再测，秒回 = **端口级阻断**确证。
- **根因**：运营商/链路对 UDP 51820 这一"知名端口"的定向丢弃。
- **处置**：换端口——服务器 `wg0.conf` 的 `ListenPort`、防火墙/安全组、客户端节点 `port`（或内核客户端 `Endpoint`）**四处同步改**；旧端口放行规则直接删除（保留无价值且扩大暴露面）。
- **教训**：部署就选非默认端口（本方案用 52800）；凡"能发不能收"，**先换端口，再怀疑实现**。

## ✦ 坑 2：SNAT vs MASQUERADE——"握手正常但只发不收"

- **症状**：`wg show` 握手正常、客户端发送字节持续上涨、接收几乎不涨（实测收 386KiB / 回 8KiB）；服务器上 tcpdump 能看到回包发出。
- **根因**：部分云平台网卡挂多个内网 IP，辅助 IP 没有公网映射；`MASQUERADE` 自动选源地址时可能选中它 → 回包进了黑洞。
- **处置**：POSTROUTING 一律写死 `SNAT --to-source <挂公网的主内网 IP>`（见 [01 文档](01-wireguard-socks.md) 关键点②）。查主 IP：云控制台网卡页，或 `ip -4 addr show <NIC>`。

## ✦ 坑 3：MSS clamp 缺失——间歇性 dial 超时 / SACK 重传

- **症状**：WG 链路配置全对，但访问 OpenAI 间歇性 `dial tcp … i/o timeout`、抓包见大量 TCP 重传（多峰分布）；移动网络尤其明显。
- **根因**：OpenAI 走 Cloudflare，SYN-ACK 携带 `mss 1400`；隧道 MTU 1420/1380 下可用载荷 1340，服务器转发时必须分片，分片在移动网络高丢包。
- **判别**：抓 SYN 看 mss 值；正常修复后所有 SYN `mss 1340`。
- **处置**：隧道 `MTU = 1380` + mangle 表 clamp（`iptables -t mangle -I FORWARD 1 -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu`），并写进 wg0.conf PostUp/PostDown 持久化。

## 坑 4：mihomo 对 WG 节点的 API 热重载副作用

- **症状**：改配置后调用 mihomo API `PUT /configs?force=true` 热重载，wireguard 节点随即 Timeout、chatgpt dial 超时；界面看配置明明"已生效"。
- **根因**：热重载对用户态 WG 出站栈有副作用（实测 1.19.x）。
- **处置**：改配置后用界面「重启内核」（`POST /restart` 重建内核）。另注意：只改增强（Merge）文件而运行时配置未重建时，热重载根本不会带上新内容。

## ✦ 坑 5：REALITY target 禁用 www.microsoft.com（OCSP 大证书）

- **症状**：**所有**客户端（mihomo/xray、各种指纹）认证握手全部 EOF；服务端 debug 日志 `REALITY: processed invalid connection: handshake did not complete successfully`；但伪装回落路径完全正常（openssl 直连能拿到真证书）——极具迷惑性，像"网络问题"。
- **根因**：microsoft 站点带 OCSP stapling，证书记录 8273 字节 > Xray REALITY 硬编码 8192 上限（XTLS/Xray-core#6356）。本机回环、干净路径、v25/v26 全版本、5 种 uTLS 指纹全部复现。
- **处置**：target/serverNames 换 `www.bing.com`（实测正常）。选 target 时避开 microsoft 系。

## 坑 6：Xray 日志属主被重置 → exit 23 循环崩溃

- **症状**：重装/升级 Xray 后服务起不来，日志 `permission denied`（exit 23）。
- **根因**：官方 install 脚本每次执行都会把 `/var/log/xray` 属主重置回 root，而服务以非 root 用户（xray）运行。
- **处置**：每次重装/升级后必做 `chown -R xray:xray /var/log/xray`；建议干脆不用官方脚本、手动解压安装（本方案如此）。

## 坑 7：xray x25519 输出格式变更（脚本静默取空值）

- **症状**：自动化脚本生成的密钥配进配置后全灭，且无报错。
- **根因**：v26 起 `xray x25519` 输出从 `Private key:` / `Public key:` 变为 `PrivateKey:` / `Password (PublicKey):` / `Hash32:`，脚本按旧字段名截取拿到空列。
- **处置**：生成后**人工逐字段核对**再写入配置；脚本解析要同时兼容两种格式。

## 坑 8：Xray 新版静默失败 → 版本锁定

- **症状**：v26.9.9 连 bing target 也握手失败，且服务端**零日志**（连坑 5 的报错都没有）。
- **处置**：生产锁 **v25.3.31**；升级必须先在非生产时段全链路回归。代理类软件（Xray/Hysteria2/mihomo）统一版本锁定纪律。

## 坑 9：Hysteria2 ACL 语法（`all` 不认、省略端口=拒全部）

- **症状**：ACL 写 `reject(all)` 服务端不认/不生效；或写 `reject(10.0.0.0/8, accept)` 之类端口语义与预期不符。
- **规则**：`reject(<cidr>)` 省略端口 = 拒绝该网段**全部端口**；`all` 不是合法网段。私网屏蔽按 [03 文档](03-hysteria2.md) 的 12 条网段逐条抄。

## 坑 10：Hysteria2 目录权限引发的静默 chown 失败 → 循环重启

- **症状**：服务 FATAL `permission denied` 循环重启；检查命令"看起来都执行过"。
- **根因**：`/etc/hysteria` 设了 750，普通用户 shell 无法展开 `*.pem` 通配符 → `chown root:hysteria /etc/hysteria/*.pem` 静默失败（shell 找不到文件），key.pem 保持 root:root → 服务读不了。`&&` 链中左侧失败不触发 set -e，极具迷惑性。
- **处置**：所有 chown/chmod 用**显式路径**（一个文件一条），不用通配符。

## 坑 11：公网明文 SOCKS5 的历史教训（安全红线）

- **症状/事故**：早期方案把 SOCKS5 直接暴露公网（NSG 全放 + 防火墙 inactive），31 个 IDC 扫描器长期挂着连接。
- **红线**：SOCKS5 只绑 WG 隧道内网地址（`10.66.66.1:1080`）；1080 **永不**加进防火墙/安全组放行；bind 永不改 `0.0.0.0`。安全三层：WG 加密 + 内网绑定 + 用户名密码。

## 坑 12：ping VPS 公网 IP 100% 丢包 ≠ 链路故障

- **症状**：`ping <VPS_IP>` 全丢，怀疑线路挂了。
- **根因**：多数云防火墙只放行业务端口（22/UDP业务端口/443），ICMP 默认不放——ping 不通是**预期行为**。
- **判别**：链路质量用 TCP 建连测试（`curl -o /dev/null -s -w '%{time_connect}\n' https://<VPS_IP>:22`）或隧道内探测，不用 ICMP。

## 坑 13：面板延迟 400~500ms 被误判为"变慢了"

- **症状**：Clash 面板里 WG 节点延迟 489ms，商业机场 50ms，怀疑"隧道劣化了"。
- **根因**：Clash 测速 = HTTP 请求经隧道 3~5 个 RTT 的叠加（移动→日本路由 ~128ms/RTT 属正常水平）；SOCKS 节点还多一层握手。商业节点 50ms 是优化路由的 1~2 RTT，不可比。
- **判别**：隧道固有 RTT 用 ICMP/UDP 到隧道内地址直接量（如 ~128ms）；**OpenAI 用途对 500ms 不敏感（TCP 长连接摊薄 RTT），稳定性 > 延迟**。判定故障看信号位（401/403）与真实调用，不看面板数字。

---

## 附：一次"全断"排查的标准顺序

```
1. TCP/SSH 到 VPS 通不通？           不通 → 网络/云平台问题
2. wg show 有握手吗？                无   → 客户端密钥/Endpoint/防火墙
3. 有握手但只发不收？                 → 坑 2（SNAT）
4. 能发不能收（服务器有回包记录）？    → 坑 1（端口阻断）→ 换端口
5. 握手正常但 dial 超时/SACK 重传？   → 坑 3（MSS clamp）
6. 换过配置才坏的？                   → 坑 4（热重载）→ 重启内核
```
