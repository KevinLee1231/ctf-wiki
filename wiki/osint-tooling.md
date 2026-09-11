---
type: tooling
tags: [osint, tooling, tools, environment]
skills: [ctf-osint]
---

# OSINT Tooling

本页服务公开账号、媒体、地点、域名、网页历史与来源关联。当前会话的网页检索、浏览器和连接器按实际可调用接口选择，不从旧工具表猜测服务已登录或 API 配额可用。

## 平台与当前入口

网页与公开数据整理优先在 Windows，Python 为 `D:/文档/新建文件夹/venv/Scripts/python.exe`（3.14.3）。已有 WSL 独立 CLI 可处理相应查询，不为普通 HTTP、图片或解析工作重建 `ctf-tools` 包集合。

| Windows 工具 | 当前状态 | 用途 |
|---|---|---|
| `requests` | venv `2.33.1` | 少量公开页面/API 请求和可复现响应记录 |
| `beautifulsoup4` / `lxml` | venv `4.14.3` / `6.0.4` | 已取得 HTML 的结构提取 |
| `Pillow` | venv `12.2.0` | 本地图像尺寸、像素及基础 EXIF |
| `pandas` | venv `3.0.2` | 账号、时间、域名和来源表关联 |
| curl | `C:/Windows/System32/curl.exe`，`8.21.0` | HTTP 状态、响应头与下载诊断 |

| 已有 WSL CLI | 绝对路径与 APT 版本 | 用途 |
|---|---|---|
| whois | `/usr/bin/whois`，`5.6.6` | 注册信息；结果受注册局脱敏与服务状态影响 |
| dig | `/usr/bin/dig`，`bind9-dnsutils 1:9.20.27-2` | 指定记录与解析器的 DNS 查询 |
| exiftool | `/usr/bin/exiftool`，`13.55+dfsg-1` | 已取得照片/媒体的详细元数据 |
| sherlock | `/usr/bin/sherlock`，`0.16.0-3` | 公开用户名候选发现；CLI 帮助可用，命中需人工核实 |

Shodan 等专项 Python 客户端、独立 OCR/图片命令没有作为本页现成入口配置；有 API 凭据不等于客户端已安装，也不要求为了一个 HTTP 查询安装 SDK。

## 完整调用

替换为题目授权范围内的域名、账号和本地文件；用户名跨站调查先限制站点和请求预算：

```pwsh
& "C:/Windows/System32/curl.exe" --head --max-time 10 "https://example.com/"
& "D:/文档/新建文件夹/venv/Scripts/python.exe" "D:/题目路径/correlate_sources.py"
wsl -d kali-linux --exec /usr/bin/dig +short TXT example.com
wsl -d kali-linux --exec /usr/bin/whois example.com
wsl -d kali-linux --exec /usr/bin/exiftool "/mnt/d/题目路径/photo.jpg"
wsl -d kali-linux --exec /usr/bin/sherlock --help
```

APT CLI 直接执行，不在 `conda run` 内调用。网页需记录来源 URL、抓取时间、原始响应及结论依据；照片 metadata 和用户名命中仅为关联线索，不能直接证明地点、拍摄者或账号归属。

## 失败处理与下一跳

- 403、验证码、登录墙、限速：保存请求条件，换可访问的原始公开来源或网页存档，不把自动化失败理解为信息不存在。
- DNS/注册信息无结果：确认记录类型、解析器、缓存与查询时间；历史网页与 DNS 见 [web-and-dns.md](web-and-dns.md)。
- 图像缺 EXIF：改用可核对地标、视角、文字与时间线；查 [geolocation-and-media.md](geolocation-and-media.md)。
- 用户名撞名：核对跨平台唯一标识、历史内容与关系证据；查 [osint-account-public-media-correlation.md](osint-account-public-media-correlation.md)。
- 任务进入服务漏洞或主机攻击链时，分别转 [web-first-pass-triage-and-chain-patterns.md](web-first-pass-triage-and-chain-patterns.md)、[pentest-attack-chains-and-tunneling.md](pentest-attack-chains-and-tunneling.md)。
- 缺包先评估 Windows 兼容版本；只有确需 Linux 能力才考虑 WSL。WSL 新增、升级、重装须先说明用途、平台必要性、目标和影响并取得明确同意。
