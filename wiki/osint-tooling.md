---
type: tooling
tags: [osint, tooling, tools, environment]
skills: [ctf-osint]
---

# OSINT Tooling

本页是 `ctf-osint` 方向本机工具信息的唯一权威来源，维护当前安装状态、版本、路径、环境、完整调用、适用边界和失败处理。`ctf-osint/SKILL.md` 只说明何时选择某类工具或知识页，不复制本页细节。

本页只描述当前真实状态；实际环境与本文不一致时，直接修正文中现状，不在本页累积核验记录或旧版本历史。公开事实的时效性仍须在每道题中实时验证。

## 完整调用约定

系统 CLI 用绝对路径从 `pwsh` 调用 WSL；Python 查询脚本使用 `ctf-tools`：

```pwsh
wsl /usr/bin/exiftool /path/to/photo.jpg
wsl /usr/bin/whois example.com
wsl /usr/bin/dig example.com A
wsl /home/kali/miniforge3/bin/conda run --no-capture-output -n ctf-tools python /path/to/osint_query.py
```

所有在线查询都应同时保存查询条件、来源 URL、响应或快照和查询时间；命令可执行不等于结论仍然有效。

## 工具选择边界

### 入口选择

- OSINT 首轮先保存原始证据：URL、图片、用户名、时间戳、坐标、页面快照。
- 本机工具主要用于辅助解析和记录；真正证据通常来自公开页面，需要保留可复查来源。
- 涉及现实人物或最新公开信息时，应实时搜索验证，不沿用旧结论。

### 不应进入 OSINT 工具链的情况

- 需要绕权限、打接口、利用漏洞或登录后访问时，转 Web/Pentest，不把技术利用写成 OSINT。
- 线索已经变成文件恢复、metadata 提取、PCAP 或内存证据时，转 Forensics。
- 结论依赖最新公开信息时，必须实时验证，不能只复用旧 raw 的结论。

### 补工具经验的触发条件

- raw 给出地理定位链，且需要固定记录地貌、路牌、建筑、天气或太阳方位证据。
- username/email/头像/社交图谱能形成可迁移的账号 pivot 流程。
- Wayback、DNS history、certificate transparency、GitHub commits 成为可复查证据链。

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

## 失败信号与转向

- 搜索结果无法复现或页面已变化：保存查询语句、搜索引擎、时间和快照 URL，再用 [web-and-dns.md](web-and-dns.md) 的历史页面/DNS 路线交叉验证。
- 线索变成图片、音频、视频或文件 metadata：不要继续靠搜索猜测，转 [geolocation-and-media.md](geolocation-and-media.md) 或对应 Forensics 媒体页。
- 需要登录、绕权限、打接口或利用服务漏洞：这已经超出 OSINT 工具页，转 [web-first-pass-triage-and-chain-patterns.md](web-first-pass-triage-and-chain-patterns.md) 或 [pentest-attack-chains-and-tunneling.md](pentest-attack-chains-and-tunneling.md)。
- 用户名枚举命中过多：先用头像、邮箱、主页互链、时间线和平台 ID 缩小身份链，再回到 [osint-account-public-media-correlation.md](osint-account-public-media-correlation.md)。

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
| **dig** | `/usr/bin/dig` | 9.20.26 | DNS 查询（bind9-dnsutils） | `dig -t any target.com` |
| **nmap** | `/usr/bin/nmap` | 7.99 | 端口/服务扫描 | `nmap -sV target` |
| **exiftool** | `/usr/bin/exiftool` | 13.55 | 图像/文件元数据 | `exiftool photo.jpg` |
| **curl** | `/usr/bin/curl` | 8.21.0 | HTTP 请求 | `curl -v https://target.com` |
| **convert** | `/usr/bin/convert` | 7.1.2-27 | ImageMagick 图像处理 | `convert image.jpg -resize 50% out.jpg` |
| **sherlock** | `/usr/bin/sherlock` | 0.16.0 | 用户名跨平台枚举 | `sherlock username` |
