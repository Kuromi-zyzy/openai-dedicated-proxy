# OpenAI Dedicated Proxy — 为 Codex / ChatGPT / OpenAI API 搭建专用代理

一套在生产环境长期稳定运行过的 **OpenAI 生态专用代理**完整方案：任意 VPS 皆可部署，覆盖服务器侧与客户端全流程，并且**整个部署过程可以直接交给 AI agent 执行**。

- **主力线路**：WireGuard 隧道 + 服务器侧 SOCKS5（microsocks）
- **容灾线路**：Xray (VLESS + REALITY Vision, TCP 443)
- **第三保险**：Hysteria2 (QUIC/UDP 443，可与 Xray 同机共存)
- **客户端**：Clash Verge (mihomo)，手动策略组一键切换多条线路

> **Disclaimer**：本项目仅供个人学习研究及合法网络访问用途，请遵守你所在地区的法律法规。内容按"现状"提供，不附带任何担保；部署产生的费用与合规责任由部署者自行承担。

---

## 从这里开始（人只做三件事，之后全部交给 AI）

### 第 1 步 · 准备一台 VPS

**一台 VPS 就能跑全部三个协议**（Ubuntu 22.04+ / Debian 12，x86_64，1C/1G 起步绰绰有余；三个协议端口互不冲突，内存占用合计 < 100MB）。本教程不展开选购，只有两点要求：

- 能开放 **TCP 22**（管理）、**一个自定义 UDP 端口**（WireGuard 用）、**TCP 443 的 TCP+UDP**（Xray / Hysteria2 用）——在你的云防火墙/安全组里放行；
- 想要**地域级容灾**（主力机被断时换另一条完全独立的线路）时，再准备第二台**不同地域**的 VPS——这是可选项，不是必需。

### 第 2 步 · SSH 引导（人做完这一次，后面都是 AI 的活）

在你自己的电脑上（Windows PowerShell / macOS / Linux 命令相同，Windows 10+ 自带 `ssh`）：

```bash
# ① 生成一对专用密钥（一路回车，passphrase 可留空）
ssh-keygen -t ed25519 -f ~/.ssh/vps_proxy -C "proxy-setup"

# ② 把公钥装到服务器上（三选一）
#    a) 供应商控制台支持贴公钥 → 直接粘贴 ~/.ssh/vps_proxy.pub 内容
#    b) 有初始密码 → 一条命令上传：
ssh-copy-id -i ~/.ssh/vps_proxy.pub root@<VPS_IP>
#    c) 或手动追加：
cat ~/.ssh/vps_proxy.pub | ssh root@<VPS_IP> "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"

# ③ 配一个短别名（编辑 ~/.ssh/config，没有就新建）：
#    Host proxy-jp
#      HostName <VPS_IP>
#      User root
#      IdentityFile ~/.ssh/vps_proxy

# ④ 验证：能打印 ok 和服务器公网 IP 即成功
ssh proxy-jp "echo ok && curl -s https://checkip.amazonaws.com"
```

### 第 3 步 · 交给 AI

对你的 AI agent（Claude Code / Codex CLI / ZCode / opencode 等均可）说一句话即可：

> 请阅读 [AGENT_RUNBOOK.md](AGENT_RUNBOOK.md) 并严格执行。SSH 别名 `proxy-jp`，部署档位 A（档位含义见下）。

Agent 会按 runbook 完成：信息收集 → 环境侦察 → 服务器侧部署（含防火墙）→ 验收 → **产出一份可直接粘贴进 Clash Verge 的客户端配置** 和凭据清单。你只需要在 Clash Verge 里做几次粘贴（GUI 操作见 [docs/04](docs/04-clash-verge-client.md)）。

> ⚠️ Agent 交付的凭据清单（私钥/密码）请保存到本地安全位置，**不要发群聊、不要截图**。

---

## 先说结论：这套方案怎么用

1. **专用，不是通用**。它只服务 OpenAI 生态（Codex CLI / ChatGPT / OpenAI API——三者同一家，风控关联出口 IP，必须**同一个出口**且长期稳定）。视频、Steam、系统更新等大流量**不要**走这条隧道：云 VPS 按出站流量计费，免费额度（如每月 100GB）扛不住。
2. **手动策略组，绝不自动漂移**。Codex agent 任务动辄十几分钟 TCP 长连接，策略组用 `select`（手动点），禁止 `url-test`/`load-balance`/`fallback`。
3. **稳定性优先级**：`长连接稳定 > 出口固定 > 易维护 > 延迟 > 峰值带宽`。推论：Clash 面板里隧道节点显示 400~500ms **是正常的**（测速 = 3~5 个 RTT 叠加），不是故障；商业机场 50ms 不可比。
4. **版本锁定**：Xray 锁 `v25.3.31`，Hysteria2 锁 `v2.12.2`——代理软件大版本升级常有静默回归，升级必须先在非生产时段验证。
5. **端口千万别用 51820**（WireGuard 默认端口）：国内链路对它存在**定向阻断**（2026-09 实测"能发不能收"持续 15 小时，换端口立即恢复，判别方法见 [排坑手册坑 1](docs/05-troubleshooting.md)）。

## 总体架构：一台 VPS 跑三个协议

```
                    ┌───────────────────────────────┐
                    │        Clash Verge (mihomo)   │  Windows
                    │  「OpenAI」手动 select 组        │
                    └───┬─────────┬─────────┬───────┘
                        ▼         ▼         ▼
      ┌─────────────── 你的 VPS（一台即可）────────────────┐
      │                                                  │
      │  WireGuard          Xray              Hysteria2  │
      │  UDP 52800          TCP 443           UDP 443    │
      │  (勿用51820!)       VLESS+REALITY     QUIC 伪装   │
      │     │               伪装 bing.com    伪装 bing.com│
      │     ▼                                            │
      │  microsocks SOCKS5 :1080（仅 WG 隧道内网）         │
      └────────────────────────┬─────────────────────────┘
                               ▼  出口 = VPS 公网 IP（三线路同出口）
```

三个协议共用同一个出口 IP，互为备份，在 Clash「OpenAI」组里手动一键切换：

- WG 端口被定向阻断 → 切 **REALITY**（TCP 443，与正常 HTTPS 无法区分）
- TCP 被干扰 / 线路高丢包 → 切 **Hysteria2**（QUIC 扛丢包）
- 想要**地域级容灾**（机器/线路整体故障） → 进阶：把 REALITY + HY2 放到第二台不同地域 VPS（部署步骤完全相同）

## 推荐档位

| 档位 | 内容 | 需要的 VPS | 适合 |
|------|------|-----------|------|
| **A（推荐起步）** | WireGuard + SOCKS5 主力线路 | 1 台 | 日常 Codex/ChatGPT，完全够用 |
| **B** | A + Xray REALITY（**与 WG 同机**） | 1 台 | 主线路端口被盯时一键切换 |
| **C（全配）** | B + Hysteria2 第三保险（**同样同机**） | 1 台 | 再加抗丢包的 QUIC 兜底 |
| 进阶（可选） | 把 REALITY + HY2 挪到第二台**不同地域** VPS | 2 台 | 地域级容灾：单机挂了仍有一条独立线路 |

**如果你懒：只部署档位 A 也完全成立**，随时可以补 B/C——三个协议装在同一台上互不冲突，共用同一个 Clash 策略组，故障时在界面点一下就切换。单机方案损失的是"地域容灾"（机器挂了全断），协议层的容灾是完整的。

三条线路的定位差异：

- **WireGuard**：配置最少、两端都是内核级实现（服务器内核态转发 + 可选 Windows 内核态客户端），延迟最低；缺点是 UDP 特征明摆着，端口被盯上就需要换端口（有成熟的判别+迁移流程）。
- **Xray REALITY**：TLS 指纹级伪装（借用真实网站的证书握手），TCP 443 与正常 HTTPS 流量无法区分，抗封锁最强；配置项稍多。
- **Hysteria2**：QUIC/UDP 443，自带前向纠错，**高丢包链路上体验最好**（劣化窗口的兜底）；部分运营商对 UDP 有 QoS。

## 客户端与服务端职责

| 层 | 职责 | 本方案对应 |
|----|------|-----------|
| 客户端策略层 | 决定"哪些流量走哪条线路" | Clash Verge 规则：OpenAI 域名 → `OpenAI` 组 |
| 客户端节点层 | 提供到服务器的加密通道 | 四种节点写法（socks5/wireguard/vless/hysteria2） |
| 服务端入口层 | 接住客户端流量 | WG 52800/udp、Xray 443/tcp、HY2 443/udp |
| 服务端出口层 | 固定出口 + 收紧安全边界 | SNAT 固定出口 IP；block 云元数据/私网 |

## 端口布局

| 端口 | 协议 | 用途 | 对外放行 |
|------|------|------|---------|
| 22 | TCP | SSH 管理 | 是（建议收敛源 IP） |
| 52800 | UDP | WireGuard | 是（**勿用 51820**） |
| 443 | TCP | Xray REALITY | 是 |
| 443 | UDP | Hysteria2 | 是 |
| 1080 | TCP | SOCKS5（仅 WG 隧道内网） | **永远不放行** |

## 安全注意事项（红线，违反即出事故）

1. **SOCKS5 只绑 WG 隧道内网地址**（`10.66.66.1:1080`），绝不绑 `0.0.0.0`、绝不加防火墙放行——明文 SOCKS5 暴露公网会被 IDC 扫描器几分钟内挂满连接（血泪教训）。
2. **出口侧必须屏蔽云元数据与私网**：Xray 用 routing rules block `169.254.169.254`（云角色凭据就在这）+ 全部 RFC1918；Hysteria2 用 `acl.inline` reject 同等网段。拿到你代理凭据的人不应能经由你的出口访问这些地址。
3. **凭据只存两处**：服务器配置文件 + 客户端 profile。不进 git、不进笔记、不进聊天记录。
4. **不动 sshd**：部署全程不修改 OpenSSH 配置、不删除已有密钥——防止把自己锁在门外。
5. **增量放行防火墙**：只添加本方案需要的规则，绝不整体关闭防火墙。

## 推荐部署顺序

```
A：VPS 装 WG+SOCKS → 客户端配节点 → 验收（出口 IP / 401 信号位）
B：同一台装 Xray REALITY → 加节点进组 → 验收
C：同一台再装 Hysteria2 → 加节点进组 → 验收（同窗三线对照）
进阶：REALITY/HY2 迁到第二台不同地域 VPS（步骤相同）
```

每一步的验收标准都写在对应文档与 runbook 里，全部通过再进入下一步。

## 文档导航

| 文档 | 内容 |
|------|------|
| [AGENT_RUNBOOK.md](AGENT_RUNBOOK.md) | **AI agent 执行手册**（从 SSH 到交付的全流程 runbook，自包含） |
| [docs/01-wireguard-socks.md](docs/01-wireguard-socks.md) | 主力线路原理与细节：SNAT/MSS clamp/端口选择三大坑 |
| [docs/02-xray-reality.md](docs/02-xray-reality.md) | REALITY 版本锁定、target 选型、防误诊 |
| [docs/03-hysteria2.md](docs/03-hysteria2.md) | 自签证书锁指纹、ACL 屏蔽私网 |
| [docs/04-clash-verge-client.md](docs/04-clash-verge-client.md) | 客户端：节点写法、手动组、热重载的坑、验证方法 |
| [docs/05-troubleshooting.md](docs/05-troubleshooting.md) | **排坑手册**：13 个实战坑的"症状→判别→根因→处置" |
| [examples/](examples/) | 服务端配置模板 + Clash 客户端 merge 片段（全部占位符） |

## 项目结构

```
openai-dedicated-proxy/
├── README.md                  # 人类向总览（本文件）
├── AGENT_RUNBOOK.md           # AI agent 执行手册
├── LICENSE
├── docs/
│   ├── 01-wireguard-socks.md
│   ├── 02-xray-reality.md
│   ├── 03-hysteria2.md
│   ├── 04-clash-verge-client.md
│   └── 05-troubleshooting.md
└── examples/
    ├── wg0.conf.example
    ├── microsocks-wg.service.example
    ├── microsocks-wg.env.example
    ├── xray-server.example.jsonc
    ├── hysteria2-server.example.yaml
    ├── hysteria2-server.service.example
    └── clash/
        ├── nodes-merge.example.yaml
        └── groups-rules.example.yaml
```
