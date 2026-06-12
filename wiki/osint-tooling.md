---
type: tooling
tags: [osint, tooling, tools, environment]
skills: [ctf-osint]
updated: 2026-05-21
---

# OSINT Tooling

本页记录 `ctf-osint` 方向的本机工具清单、调用层、路径和适用边界。`SKILL.md` 只保留首轮工具摘要；需要详细路径、环境和专项工具说明时读取本页。

## 调用层与覆盖状态

### 非交互调用原则

- OSINT 首轮先保存原始证据：URL、图片、用户名、时间戳、坐标、页面快照。
- 本机工具主要用于辅助解析和记录；真正证据通常来自公开页面，需要保留可复查来源。
- 涉及现实人物或最新公开信息时，应实时搜索验证，不沿用旧结论。

### 知识页覆盖状态

- 当前覆盖地理/媒体、社交媒体、Web/DNS OSINT 三条主线，页面数量偏少但入口清晰。
- 后续应补 GitHub/代码仓库 OSINT、地图/街景推理、历史网页、用户名跨平台聚合和元数据反查。

### 后续补强方向

- Geolocation checklist：地貌、路牌、建筑、天气、太阳方位。
- Account pivot：username/email/头像/社交图谱。
- Web history：Wayback、DNS history、certificate transparency、GitHub commits。

## 本机工具清单（按使用时机）

### 首轮常用

| 工具 | 为什么放在首轮 |
|---|---|
| `exiftool` | 图片和文档线索的低成本首检 |
| `whois` / `dig` | 域名与基础设施线索的第一跳 |
| `curl` | 取页面、API 和原始响应 |

### 专项按需

- 网络与基础设施：`shodan`、`nmap`
- 程序化查询：`dnspython`、`requests`
- 图像辅助：`Pillow`、`convert`
- 用户名枚举：`sherlock`

### 当前未装 / 建议按需补装

- 当前没有明显高优先级 CLI 缺口。`WhatsMyName` 现在更适合作为数据源 / 网页核对，而不是默认本地命令行入口。

## 详细清单

### ctf-tools conda 环境

| 工具 | 版本 | 功能 | 典型用法 |
|---|---|---|---|
| **shodan** | 1.31.0 | Shodan API 搜索 | `shodan search "hostname:target.com"` |
| **dnspython** | 2.8.0 | DNS 查询 | `dns.resolver.resolve("target.com", "A")` |
| **Pillow** | 11.3.0 | 图像元数据分析 | `from PIL import Image; Image.open("img.jpg")` |
| **requests** | 2.33.1 | HTTP/API 请求 | `requests.get("https://api.example.com")` |

### 系统全局命令（WSL Kali）

| 工具 | 路径 | 版本 | 功能 | 典型用法 |
|---|---|---|---|---|
| **whois** | `/usr/bin/whois` | 5.6.6 | WHOIS 查询 | `whois target.com` |
| **dig** | `/usr/bin/dig` | 9.20.22 | DNS 查询（bind9-dnsutils） | `dig -t any target.com` |
| **nmap** | `/usr/bin/nmap` | 7.99 | 端口/服务扫描 | `nmap -sV target` |
| **exiftool** | `/usr/bin/exiftool` | 13.50 | 图像/文件元数据 | `exiftool photo.jpg` |
| **curl** | `/usr/bin/curl` | 8.19.0 | HTTP 请求 | `curl -v https://target.com` |
| **convert** | `/usr/bin/convert` | 7.1.2 | ImageMagick 图像处理 | `convert image.jpg -resize 50% out.jpg` |
| **sherlock** | `/usr/bin/sherlock` | 0.16.0 | 用户名跨平台枚举 | `sherlock username` |
