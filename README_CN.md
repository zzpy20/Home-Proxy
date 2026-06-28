# Home-Proxy

[English](README.md) | [中文](README_CN.md)

在布里斯班家庭设备（Mac Mini 或 Apple TV）上运行的 VLESS+Reality 代理——利用住宅 IP 实现终极抗 GFW 方案。

> **状态：** 参考架构，暂未实际部署——现有方案（Linode 东京的 VLESS-Reality + 阿里云深圳→新加坡中转）已能满足当前需求。本文档作为 VPS 代理不可靠时的未来备选方案存档。

**相关项目：**
- [VLESS-Reality](https://github.com/zzpy20/VLESS-Reality) — Linode 东京的 VLESS+Reality（主要代理）
- [Alan-Infrastructure](https://github.com/zzpy20/Alan-Infrastructure) — Shadowsocks 深圳→新加坡中转（备用）

---

## 为什么住宅 IP 更优？

基于 VPS 的代理（Linode、阿里云）使用的是已知商业数据中心的 IP。GFW 可以将任意 IP 与公开的 ASN 地址段交叉比对，无论运行什么协议，都能识别出数据中心来源。

布里斯班家庭 IP 由住宅 ISP（如 Aussie Broadband、TPG）分配，具备以下特点：
- **与普通家庭流量无法区分** — GFW 看到的是一个澳大利亚家庭连接，而非代理服务器
- **不在任何数据中心 ASN 地址段内** — 不存在可被标记的 ASN 不匹配
- **无法大规模封锁** — GFW 若要封锁，就必须封锁所有澳大利亚住宅 ISP 流量，在政治上不可能

这使其成为**比任何 VPS 方案都更可靠、更难被封锁的选择**，且除电费外无额外成本。

---

## 架构

```
中国设备 ──VLESS+Reality──▶ yourhome.duckdns.org（布里斯班 NBN）──▶ 互联网
                                       │
                               Mac Mini / Apple TV
                               （sing-box，常开）
```

- 协议：VLESS + XTLS-Vision + Reality（与 VLESS-Reality 相同）
- SNI 伪装：`itunes.apple.com`（或任意支持 TLS 1.3 的非 CDN 域名）
- 端口：443
- DNS：DuckDNS（免费动态 DNS——自动追踪家庭 IP 变化）

---

## 为什么更可靠

| | VLESS-Reality（Linode） | Home-Proxy（布里斯班） |
|---|---|---|
| IP 类型 | 商业数据中心（Linode ASN） | 住宅 ISP |
| ASN 不匹配风险 | 有——Linode 不是 Apple 基础设施 | 无——看起来就是真实家庭网络 |
| GFW 封锁风险 | IP 层封锁可能发生 | 几乎为零——无法批量封锁住宅 ISP |
| 稳定性 | 数据中心级别 | 取决于家庭电力和 NBN |
| 带宽 | 1Gbps+ VPS | NBN 上行（典型 20–50 Mbps） |
| 成本 | 约 $5–10/月 | 仅电费 |
| 被封后恢复 | 新建 VPS + replace.sh（约 10 分钟） | 重启家庭路由器 → 获得新 IP，DuckDNS 5 分钟内自动更新 |

---

## 前置条件

- Mac Mini 或 Apple TV（常开，连接布里斯班 NBN）
- 家庭路由器将 443 端口转发至 Mac Mini 本地 IP
- [DuckDNS](https://www.duckdns.org) 账号——免费动态 DNS
- Mac Mini 已安装 Docker
- 可从中国通过 DuckDNS 域名 SSH 访问 Mac Mini

---

## 部署步骤

### 1. DuckDNS — 动态 DNS

家庭 NBN IP 会偶尔变动。DuckDNS 自动监测并更新域名解析。

1. 访问 [duckdns.org](https://www.duckdns.org)，用 Google 账号登录
2. 创建子域名，如 `alanhome.duckdns.org`
3. 在 Mac Mini 上安装 DuckDNS 更新程序——每 5 分钟自动更新：
```bash
# Mac Mini 上的 cron 任务
*/5 * * * * curl -s "https://www.duckdns.org/update?domains=alanhome&token=YOUR_TOKEN&ip=" > /dev/null
```

### 2. 路由器——端口转发

在家庭路由器设置中转发：
- 外部端口 443 → Mac Mini 本地 IP，端口 443
- 外部端口 22 → Mac Mini 本地 IP，端口 22（供从中国 SSH 连接）

### 3. Mac Mini 安装 Docker

```bash
# 从 docker.com 下载 Docker Desktop，或通过 Homebrew 安装：
brew install --cask docker
```

### 4. 部署 sing-box

克隆 [VLESS-Reality](https://github.com/zzpy20/VLESS-Reality) 并运行相同流程——只需将目标指向 Mac Mini 而非远程 VPS：

```bash
git clone https://github.com/zzpy20/VLESS-Reality.git
cd VLESS-Reality
bash generate-keys.sh       # 生成 Reality 密钥对 + UUID
bash deploy.sh              # 启动 sing-box 容器
```

### 5. Shadowrocket 配置

与 VLESS-Reality 相同，仅替换服务器域名：

| 字段 | 值 |
|---|---|
| 服务器 | `alanhome.duckdns.org` |
| 端口 | 443 |
| UUID | *(来自 .env)* |
| Flow | xtls-rprx-vision |
| 安全 | reality |
| SNI | itunes.apple.com |
| 指纹 | chrome |
| Public Key | *(来自 .env)* |
| Short ID | *(来自 .env)* |

---

## 注意事项

- **上行带宽是瓶颈** — NBN 上行（20–50 Mbps）即为代理带宽上限。单人使用足够，4K 串流可能偏紧。
- **无数据中心级别稳定性** — 家庭断电或 NBN 故障即代理中断，不像 VPS 有冗余保障。
- **家庭 IP 仍可能被单独封锁** — 对于单一低流量连接极不可能发生。即便发生，GFW 只会封锁从中国到该 IP 的流量；你的 ISP 对此毫不知情，布里斯班的邻居也完全不受影响。恢复方法：远程重启家庭路由器/猫（让家人操作，或使用智能插座）——ISP 会分配新 IP，DuckDNS 在 5 分钟内自动更新，Shadowrocket 无需任何改动。
- **443 端口冲突** — 若 Mac Mini 上有其他服务占用 443 端口，需重新映射。

---

## 在中国时的优先顺序

1. **VLESS-Reality**（Linode 东京）— 首选，速度快，中日延迟低
2. **Alan-Infrastructure**（深圳→新加坡中转）— Linode IP 被封时的备用
3. **Home-Proxy**（布里斯班）— 抗 GFW 能力最强，但中澳延迟较高，且依赖家庭网络稳定性
4. **ss-server-config** — 最后手段

> Home-Proxy 排在第 3 位，并非因为抗 GFW 能力弱，而是因为中国→澳大利亚的延迟高于中国→日本，且家庭网络稳定性不如 VPS。
