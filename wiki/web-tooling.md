---
type: tooling
tags: [web, tooling, tools, environment]
skills: [ctf-web]
---

# Web Tooling

本页维护 HTTP/API、认证、浏览器、模板和服务端输入边界的本机工具。先记录正常请求与响应，再根据具体差异选择有界验证。

## Windows 默认入口

Python 脚本固定使用 `D:/文档/新建文件夹/venv/Scripts/python.exe`（3.14.3）。HTTP、JSON、JWT、Flask session 与页面解析不需要进入 WSL `ctf-tools`。

| 工具 | 当前版本与位置 | 用途 |
|---|---|---|
| requests | Windows venv `2.33.1` | 请求重放、会话、Cookie 和超时控制 |
| PyJWT | Windows venv `2.12.1` | JWT header/payload 与签名算法实验 |
| Flask / itsdangerous / Jinja2 | Windows venv `3.1.3` / `2.2.0` / `3.1.6` | 按题目配置复现 session、签名与模板语义 |
| beautifulsoup4 / lxml | Windows venv `4.14.3` / `6.0.4` | HTML/XML 结构提取 |
| pycryptodome / cryptography | Windows venv `3.23.0` / `46.0.7` | 已确定的加密、签名与证书流程 |
| curl | `C:/Windows/System32/curl.exe`，`8.21.0` | 原始 HTTP、响应头和连接诊断 |
| Burp Suite | `D:/CTF工具/BurpSuite/burpsuite.jar`，文件存在 | GUI 代理历史、Repeater 和手工请求修改 |
| ysoserial | `D:/CTF工具/ysoserial-all.jar`，文件存在 | 已指向 Java 反序列化时的专项候选；未验证当前 JDK 与具体 gadget 兼容性 |

Windows 存在 `httpx2` 发行包，但不能把它当成任意旧脚本的 `httpx` API 保证；普通请求可先用已验证的 requests。当前没有 flask-unsign、jwcrypto、hashpumpy、hash_extender 或 phpggc 调用入口；需要专门能力时先确认漏洞机制，再评估 Windows 最小实现或兼容新版本。

```pwsh
& "C:/Windows/System32/curl.exe" --include --max-time 10 "http://127.0.0.1:8080/"
& "D:/文档/新建文件夹/venv/Scripts/python.exe" -c 'import requests; r=requests.get("http://127.0.0.1:8080/", timeout=10, allow_redirects=False); print(r.status_code); print(dict(r.headers)); print(r.text[:1000])'
& "D:/文档/新建文件夹/venv/Scripts/python.exe" "D:/题目路径/replay_request.py"
```

示例 URL 替换为题目授权端点。JWT 解析结果不是鉴权结论；Flask/Jinja2 行为要对齐题目 secret、salt、算法和实际模板上下文，不要求为了复用旧脚本保留已清理版本。

## Burp 与浏览器

当前会话没有暴露 Burp MCP。目录中 `mcp-proxy.jar` 存在不等于插件已加载或连接成功，不从旧固定工具名列表构造调用。需要代理时先核对 GUI、监听地址、浏览器代理与本题范围，再查看正常请求历史；重复请求优先使用 Repeater 并保留原始报文。

在需要显示 Burp GUI 时，由 `pwsh` 直接启动现有 jar：

```pwsh
& "C:/Program Files/Eclipse Adoptium/jdk-21.0.10.7-hotspot/bin/java.exe" -jar "D:/CTF工具/BurpSuite/burpsuite.jar"
```

上述为文件入口，不代表本次已验证 GUI 启动或扩展加载。后续若发现可调用 MCP，先核对参数 schema、会话及范围，再按“读取历史 → 选择请求 → 修改最少字段 → 比较响应”使用。不得把开启持续代理、Intruder 批量任务或更改全局配置作为一次读请求的隐含步骤。

## 已有 Kali CLI

下列 APT 工具仍保留；它们直接从 `pwsh` 调 WSL，不能套入 Conda。ffuf、gobuster、nuclei 等已编译程序运行不依赖 Go 编译器。

| 工具 | 绝对入口 | APT 版本 | 用途 |
|---|---|---|---|
| ffuf | `/usr/bin/ffuf` | `2.2.1-1` | 已知输入点的有界内容发现 |
| feroxbuster | `/usr/bin/feroxbuster` | `2.13.1-0kali3` | 目录枚举，递归深度与请求量需限制 |
| gobuster | `/usr/bin/gobuster` | `3.8.2-1` | 目录、DNS 等模式枚举 |
| sqlmap | `/usr/bin/sqlmap` | `1.10.8-1` | 已有 SQL 注入证据时进一步验证 |
| nuclei | `/usr/bin/nuclei` | `3.11.1-0kali1` | 指定模板检查；模板是否存在另行核对，不自动更新 |
| nikto | `/usr/bin/nikto` | `1:2.6.1-0kali1` | 已获准的 Web 服务配置检查 |
| cewl | `/usr/bin/cewl` | `6.2.1-1` | 按已授权站点和抓取范围构造词表 |
| searchsploit | `/usr/bin/searchsploit` | `exploitdb 20260826-0kali1` | 本地漏洞资料检索，命中仍须核对版本与条件 |
| SecLists | `/usr/share/seclists` | `2025.3-0kali1` | 现有字典；`Discovery/Web-Content/common.txt` 存在 |

不假定旧的 dirb/rockyou 路径存在。先看工具帮助与本地资料，不发起扫描：

```pwsh
wsl -d kali-linux --exec /usr/bin/ffuf -h
wsl -d kali-linux --exec /usr/bin/sqlmap --version
wsl -d kali-linux --exec /usr/bin/searchsploit "product version"
```

明确目标、输入点、字典大小和请求预算后，内容发现示例为：

```pwsh
wsl -d kali-linux --exec /usr/bin/ffuf -u "http://127.0.0.1:8080/FUZZ" -w "/usr/share/seclists/Discovery/Web-Content/common.txt" -t 2 -rate 5 -maxtime 30
```

## 失败处理与转向

- 重放与浏览器不同：对齐 Cookie、跳转、Content-Type、代理、编码与请求体；看 [web-first-pass-triage-and-chain-patterns.md](web-first-pass-triage-and-chain-patterns.md)。
- 上传或响应其实是文件容器：保留原始字节，查 [file-signatures-and-flag-artifact-hunting.md](file-signatures-and-flag-artifact-hunting.md)。
- SSRF/协议注入连到内部服务：查 [protocol-relay-and-internal-service-injection.md](protocol-relay-and-internal-service-injection.md)、[workflow-runner-internal-api-chain.md](workflow-runner-internal-api-chain.md)。
- 已取得 foothold，剩下凭据、横向或隧道：转 [pentest-attack-chains-and-tunneling.md](pentest-attack-chains-and-tunneling.md)。
- APT Python 工具报缺模块：先退出 Conda 路由并核对系统入口，不向系统 Python pip 安装。任何 WSL 新增、升级、重装须先说明 Linux 必要性、方式、环境及依赖影响并取得明确同意。

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [VNCTF2026-web-pentest-wp](../raw/web/VNCTF2026-web-pentest-wp.md) | 前端混合加密登录把 SM4 `key/iv` 经 SM2 封装给服务端；若封装值可固定复用，只需重算业务密文和 MD5 签名即可爆破。 |
