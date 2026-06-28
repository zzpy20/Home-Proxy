# Home-Proxy

[English](README.md) | [中文](README_CN.md)

VLESS+Reality proxy running on a Brisbane home machine (Mac Mini or Apple TV) — the ultimate GFW-resistant setup using a residential IP.

> **Status:** Reference architecture. Not actively deployed — existing solutions (VLESS-Reality on Linode, Alan-Infrastructure SZ→SG relay) are sufficient for current needs. This is documented as a future fallback if VPS-based proxies become unreliable.

**Related projects:**
- [VLESS-Reality](https://github.com/zzpy20/VLESS-Reality) — VLESS+Reality on Linode Tokyo (primary proxy)
- [Alan-Infrastructure](https://github.com/zzpy20/Alan-Infrastructure) — Shadowsocks SZ→SG relay (fallback)

---

## Why Residential IP?

VPS-based proxies (Linode, Alibaba Cloud) rent IPs from known commercial datacenters. GFW can cross-reference any IP against published ASN ranges and identify it as a datacenter — regardless of what protocol is running on it.

A Brisbane home IP is assigned by a residential ISP (e.g. Aussie Broadband, TPG). It is:
- **Indistinguishable from normal household traffic** — GFW sees an Australian home connection, not a proxy server
- **Not in any datacenter ASN range** — no ASN mismatch to flag
- **Impossible to block at scale** — GFW would have to block all Australian residential ISP traffic, which is politically impossible

This makes it **more reliable and harder to block than any VPS solution**, at zero ongoing cost beyond electricity.

---

## Architecture

```
Device (China) ──VLESS+Reality──▶ yourhome.duckdns.org (Brisbane NBN) ──▶ Internet
                                         │
                                   Mac Mini / Apple TV
                                   (sing-box, always on)
```

- Protocol: VLESS + XTLS-Vision + Reality (same as VLESS-Reality)
- SNI steal: `itunes.apple.com` (or any TLS 1.3, non-CDN domain)
- Port: 443
- DNS: DuckDNS (free dynamic DNS — tracks your home IP automatically)

---

## Why This Is More Reliable

| | VLESS-Reality (Linode) | Home-Proxy (Brisbane) |
|---|---|---|
| IP type | Commercial datacenter (Linode ASN) | Residential ISP |
| ASN mismatch risk | Yes — Linode is not Apple infrastructure | None — looks like a real home |
| GFW blocking risk | IP-level blocking possible | Near zero — can't bulk-block residential ISPs |
| Uptime | Datacenter-grade | Depends on home power + NBN |
| Bandwidth | 1Gbps+ VPS | NBN upload (20–50 Mbps typical) |
| Cost | ~$5–10/month | Electricity only |
| Recovery if blocked | New VPS + replace.sh (~10 min) | ISP rotates IP naturally |

---

## Prerequisites

- Mac Mini or Apple TV (always on, connected to Brisbane NBN)
- Port 443 forwarded on home router → Mac Mini local IP
- [DuckDNS](https://www.duckdns.org) account — free dynamic DNS
- Docker installed on Mac Mini
- SSH access to Mac Mini from China (via the DuckDNS hostname)

---

## Setup

### 1. DuckDNS — Dynamic DNS

Your home NBN IP changes occasionally. DuckDNS watches it and updates a hostname automatically.

1. Go to [duckdns.org](https://www.duckdns.org), log in with Google
2. Create a subdomain e.g. `alanhome.duckdns.org`
3. Install the DuckDNS updater on Mac Mini — runs every 5 minutes:
```bash
# cron job on Mac Mini
*/5 * * * * curl -s "https://www.duckdns.org/update?domains=alanhome&token=YOUR_TOKEN&ip=" > /dev/null
```

### 2. Router — Port Forward

In your home router settings, forward:
- External port 443 → Mac Mini local IP, port 443
- External port 22 → Mac Mini local IP, port 22 (for SSH access from China)

### 3. Install Docker on Mac Mini

```bash
# Download Docker Desktop from docker.com, or via Homebrew:
brew install --cask docker
```

### 4. Deploy sing-box

Clone [VLESS-Reality](https://github.com/zzpy20/VLESS-Reality) and run the same setup — just target your Mac Mini instead of a remote VPS:

```bash
git clone https://github.com/zzpy20/VLESS-Reality.git
cd VLESS-Reality
bash generate-keys.sh       # generate Reality keypair + UUIDs
bash deploy.sh              # start sing-box container
```

### 5. Shadowrocket Config

Same as VLESS-Reality — just replace the server hostname:

| Field | Value |
|---|---|
| Server | `alanhome.duckdns.org` |
| Port | 443 |
| UUID | *(from .env)* |
| Flow | xtls-rprx-vision |
| Security | reality |
| SNI | itunes.apple.com |
| Fingerprint | chrome |
| Public Key | *(from .env)* |
| Short ID | *(from .env)* |

---

## Caveats

- **Upload speed is the bottleneck** — NBN upload (20–50 Mbps) is your proxy bandwidth. Fine for one person, tight for 4K streaming.
- **No datacenter uptime** — if home power or NBN goes down, so does the proxy. Unlike a VPS, there's no redundancy.
- **Home IP can still be individually blocked** — unlikely for a single low-traffic connection, and your ISP naturally rotates it anyway.
- **Port 443 conflicts** — if anything else on your Mac Mini uses port 443, you'll need to remap.

---

## Priority Order (When in China)

1. **VLESS-Reality** (Linode Tokyo) — primary, fast, low latency to Japan
2. **Alan-Infrastructure** (SZ→SG relay) — fallback if Linode IP gets blocked
3. **Home-Proxy** (Brisbane) — most GFW-resistant, but higher latency (China→Australia) and dependent on home uptime
4. **ss-server-config** — last resort

> Home-Proxy sits at #3 not because it's less reliable against GFW, but because China→Australia latency is higher than China→Japan, and home uptime is less guaranteed than a VPS.
