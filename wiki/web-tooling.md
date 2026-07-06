---
type: tooling
tags: [web, tooling, tools, environment]
skills: [ctf-web]
updated: 2026-07-06
---

# Web Tooling

本页记录 `ctf-web` 方向的本机工具清单、调用层、路径和适用边界。`SKILL.md` 只保留首轮工具摘要；需要详细路径、环境和专项工具说明时读取本页。

## 工具选择边界

### 入口选择

- Web 首轮优先 Burp MCP + `curl`：先保存请求/响应、cookie、redirect、认证状态和差异样本。
- fuzz/扫描要有边界：确认应用面不足时才用 `ffuf/gobuster/feroxbuster`，确认注入信号后才用 `sqlmap`。
- Java/PHP 反序列化、hash length extension、JWT/JWE、PDF 附件等专项工具按本页路径进入，不要和普通请求测试混在一起。

### 不应进入 Web 工具链的情况

- 目标主要是本地 binary、内存破坏、固件或模型文件时，Web 只保留传输证据，主线转对应专项。
- 扫描器没有输入面、认证态和 baseline 时不要先跑；先用 Burp/curl 建最小差异样本。
- Web 只是进入内网、runner、主机命令或凭据链的第一步时，要及时转 Pentest/Reverse/Pwn。

### 补工具经验的触发条件

- raw 给出 H2/H1 转换、CL/TE、proxy normalization 等 HTTP parser differential。
- Browser-side 证据集中在 postMessage、XS-Leaks、CSP bypass 或 Service Worker。
- Supply chain/CI 题需要 artifact trust、workflow token、internal runner 的工具链复现。

## 本机工具清单（按使用时机）

### 首轮常用

| 工具 | 为什么放在首轮 |
|---|---|
| Burp MCP | 直接看请求/响应、认证流和参数，是 Web 题最有信息密度的入口 |
| `curl` | 快速看跳转、响应头、Cookie 和基础行为 |
| `gobuster` / `ffuf` | 当应用面还不完整时，用来补隐藏路由 |
| `sqlmap` | 只有在已出现注入信号时，才作为快速验证工具进入首轮 |

### 专项按需

- 指纹/组件验证：`nuclei`、`nikto`
- 子域与外围面：`subfinder`、`nmap`
- 认证专项：`flask-unsign`、`john`、`hashcat`
- 哈希/长度扩展：`hashpumpy`、`hash_extender`
- 流量与数据处理：`tshark`、`tcpdump`、`jq`
- 题型专项：`hydra`、`ysoserial`、`pdfdetach`、`requests`、`pwntools`、`/home/kali/phpggc/phpggc`

### 当前未装 / 建议按需补装

- 当前没有明显缺口。`jwcrypto` 已在 `ctf-tools` 中，只有在需要完整 JWT/JWS/JWE 构造、验证或调试时才把它拉进首轮流程；不要把它当成 `flask-unsign` 的替代品。

## 失败信号与转向

- Burp/curl 只有页面差异，没有明确参数控制点：先回到 [web-first-pass-triage-and-chain-patterns.md](web-first-pass-triage-and-chain-patterns.md) 建 baseline、认证态和输入面，不要直接上扫描器。
- 目录扫描命中大量静态文件或误报：收敛到路由、文件首检或源码线索；若导出附件/压缩包，转 [file-signatures-and-flag-artifact-hunting.md](file-signatures-and-flag-artifact-hunting.md)。
- `sqlmap`、nuclei 或模板扫描无结果：保留能复现差异的请求、响应片段和参数控制点，再按 SQLi、SSRF、反序列化、认证或浏览器侧 family 分流。
- 请求已经进入内网服务、runner、token 或主机命令阶段：转 [protocol-relay-and-internal-service-injection.md](protocol-relay-and-internal-service-injection.md)、[workflow-runner-internal-api-chain.md](workflow-runner-internal-api-chain.md) 或 [pentest-attack-chains-and-tunneling.md](pentest-attack-chains-and-tunneling.md)。

## 详细清单

### Burp Suite MCP（第一优先）

| 工具 | 功能 | 调用方式 |
|---|---|---|
| `burp__base64_encode/decode` | Base64 编解码（cookie/token/payload） | MCP 直接调用 |
| `burp__url_encode/decode` | URL 编解码 | MCP 直接调用 |
| `burp__get_proxy_http_history` | 查看 HTTP 代理历史 | MCP，`count`/`offset` 参数 |
| `burp__get_proxy_http_history_regex` | 按正则匹配 HTTP 历史 | MCP，`regex` 参数 |
| `burp__send_http1_request` | 发送 HTTP/1.1 请求 | MCP，`content`+`targetHostname`+`targetPort`+`usesHttps` |
| `burp__send_http2_request` | 发送 HTTP/2 请求 | MCP，`headers`+`pseudoHeaders`+`requestBody` |
| `burp__create_repeater_tab` | 创建 Repeater 标签页（手工测试） | MCP，`tabName` 可选 |
| `burp__send_to_intruder` | 发送到 Intruder（fuzzing） | MCP |
| `burp__set_proxy_intercept_state` | 启用/禁用拦截 | MCP，`intercepting: true/false` |
| `burp__generate_random_string` | 生成随机字符串 | MCP |

### 系统全局命令（WSL Kali）

| 工具 | 路径 | 功能 | 典型用法 |
|---|---|---|---|
| **gobuster** | `/usr/bin/gobuster` 3.8.2 | 目录/文件快速扫描 | `gobuster dir -u URL -w /usr/share/wordlists/dirb/common.txt` |
| **ffuf** | `/usr/bin/ffuf` 2.1.0-dev | 灵活 Web fuzzer（支持参数化 fuzz） | `ffuf -u URL/FUZZ -w wordlist.txt` |
| **feroxbuster** | `/usr/bin/feroxbuster` 2.13.1 | 递归内容发现，适合目录树较深时补强 `gobuster` / `ffuf` | `feroxbuster -u URL` |
| **nuclei** | `/usr/bin/nuclei` 3.8.0 | YAML 模板驱动漏洞扫描（CVE/配置/泄露） | `nuclei -u URL -t cves/ -t exposures/` |
| **sqlmap** | `/usr/bin/sqlmap` 1.10.4 | SQL 注入自动检测与利用 | `sqlmap -u "URL?id=1" --dbs` |
| **hydra** | `/usr/bin/hydra` 9.6 | 在线口令爆破（HTTP/FTP/SSH 等） | `hydra -l admin -P pass.txt target http-post-form "/login:user=^USER^&pass=^PASS^:F=error"` |
| **nmap** | `/usr/bin/nmap` 7.99 | 端口扫描/服务版本/脚本引擎 | `nmap -sV -sC target` |
| **subfinder** | `/usr/bin/subfinder` 2.13.0 | 被动子域名发现 | `subfinder -d target.com` |
| **nikto** | `/usr/bin/nikto` | Web 服务器检查 | `nikto -h https://target.com` |
| **tshark** | `/usr/bin/tshark` 4.6.4 | 命令行 Wireshark（PCAP 分析） | `tshark -r capture.pcap -Y "http"` |
| **curl** | `/usr/bin/curl` | HTTP 客户端（请求/响应测试） | `curl -v http://target.com` |
| **nc** | `/usr/bin/nc` | TCP/UDP 连接/netcat | `nc target 80 <<< "GET / HTTP/1.0"` |
| **hash_extender** | `/usr/local/bin/hash_extender` | CLI 哈希长度扩展攻击 | `hash_extender -d data -s hash -a append -l len` |
| **john** | `/usr/sbin/john` 1.9.0-jumbo | 哈希破解/JWT secret 暴力 | `john jwt.txt --wordlist=rockyou.txt` |
| **hashcat** | `/usr/bin/hashcat` 7.1.2 | GPU 哈希破解（全类型支持） | `hashcat -m 16500 jwt.txt rockyou.txt` |
| **jq** | `/usr/bin/jq` 1.8.1 | JSON 解析/查询/格式化 | `curl -s API \| jq '.flag'` |
| **tcpdump** | `/usr/bin/tcpdump` 4.99.6 | 网络流量抓取 | `tcpdump -i eth0 -w capture.pcap port 80` |
| **pdfdetach** | `/usr/bin/pdfdetach` 25.03.0 | PDF 附件提取 | `pdfdetach -saveall file.pdf` |

### Python 包（ctf-tools conda）

| 工具 | 版本 | 功能 | 典型用法 |
|---|---|---|---|
| **flask-unsign** | 1.2.1 | Flask session 解码/签名暴破/字典破解 | `flask-unsign --decode --cookie 'eyJ...'` |
| **jwcrypto** | 1.5.7 | JWT/JWS/JWE 构造、验证与调试 | `from jwcrypto import jwt, jwk, jws, jwe` |
| **hashpumpy** | 1.2 | 哈希长度扩展攻击 | `hashpumpy.hashpump(known_hash, known_data, append_data, key_len)` |
| **requests** | 2.33.1 | HTTP 请求库（session/cookie/代理） | `requests.get(URL, cookies={}, proxies={})` |
| **pwntools** | 4.15.0 | 数据打包/编码/序列化 | `from pwn import xor; xor(data, key)` |

### WSL 本地源码工具

| 工具 | 路径 | 功能 | 典型用法 |
|---|---|---|---|
| **phpggc** | `/home/kali/phpggc/phpggc` | PHP 反序列化 gadget chain 生成 | `/home/kali/phpggc/phpggc -l` |

### Windows 本地

| 工具 | 路径 | 功能 | 典型用法 |
|---|---|---|---|
| **ysoserial** | `D:\CTF工具\ysoserial-all.jar` | Java 反序列化 payload 生成 | `java -jar ysoserial-all.jar CommonsCollections1 'cmd'` |


---

## Burp Suite MCP 详细操作

Burp Suite 已连接为 MCP 服务器，可通过工具直接控制 Burp 功能。以下工具可在 CTF 解题中直接调用：

### 编码/解码工具

| 工具 | 功能 | 示例 |
|------|------|------|
| `mcp__burp__base64_encode` | Base64 编码 | 编码 XSS payload、JWT 片段 |
| `mcp__burp__base64_decode` | Base64 解码 | 解码 cookie、token |
| `mcp__burp__url_encode` | URL 编码 | 编码特殊字符 payload |
| `mcp__burp__url_decode` | URL 解码 | 解码 URL-encoded 参数 |
| `mcp__burp__generate_random_string` | 生成随机字符串 | 生成 fuzzing 随机输入 |

### Proxy 历史查看

| 工具 | 功能 | 参数 |
|------|------|------|
| `mcp__burp__get_proxy_http_history` | 查看 HTTP 代理历史 | `count`（数量）, `offset`（偏移） |
| `mcp__burp__get_proxy_http_history_regex` | 按正则匹配查看 HTTP 历史 | `regex`, `count`, `offset` |
| `mcp__burp__get_proxy_websocket_history` | 查看 WebSocket 代理历史 | `count`, `offset` |
| `mcp__burp__get_proxy_websocket_history_regex` | 按正则匹配查看 WebSocket 历史 | `regex`, `count`, `offset` |

```bash
# 查看最近 20 条 HTTP 历史
get_proxy_http_history(count=20, offset=0)

# 按正则搜索包含 flag、token、secret 的请求
get_proxy_http_history_regex(regex="flag|token|secret", count=20, offset=0)

# 查看 WebSocket 历史
get_proxy_websocket_history(count=20, offset=0)
```

### 请求发送工具

| 工具 | 功能 | 参数 |
|------|------|------|
| `mcp__burp__send_http1_request` | 发送 HTTP/1.1 请求 | `content`, `targetHostname`, `targetPort`, `usesHttps` |
| `mcp__burp__send_http2_request` | 发送 HTTP/2 请求 | `headers`, `pseudoHeaders`, `requestBody`, `targetHostname`, `targetPort`, `usesHttps` |

```bash
# 发送 HTTP/1.1 请求（注意使用 \r\n 作为换行）
send_http1_request(
  content="GET /admin HTTP/1.1\r\nHost: target.com\r\n\r\n",
  targetHostname="target.com",
  targetPort=443,
  usesHttps=true
)
```

### Repeater 和 Intruder

| 工具 | 功能 | 参数 |
|------|------|------|
| `mcp__burp__create_repeater_tab` | 创建 Repeater 标签页 | `content`, `targetHostname`, `targetPort`, `usesHttps`, `tabName`(可选) |
| `mcp__burp__send_to_intruder` | 发送请求到 Intruder | `content`, `targetHostname`, `targetPort`, `usesHttps`, `tabName`(可选) |

```bash
# 在 Repeater 中打开一个请求以便编辑和手动测试
create_repeater_tab(
  content="POST /api/login HTTP/1.1\r\nHost: target.com\r\nContent-Type: application/json\r\n\r\n{\"user\":\"admin\",\"pass\":\"test\"}",
  targetHostname="target.com",
  targetPort=443,
  usesHttps=true,
  tabName="Login Test"
)

# 发送到 Intruder 进行 fuzzing
send_to_intruder(
  content="GET /page?id=§1§ HTTP/1.1\r\nHost: target.com\r\n\r\n",
  targetHostname="target.com",
  targetPort=443,
  usesHttps=true
)
```

### 编辑器操作

| 工具 | 功能 |
|------|------|
| `mcp__burp__get_active_editor_contents` | 获取当前活动编辑器的内容 |
| `mcp__burp__set_active_editor_contents` | 设置当前活动编辑器的内容 |

### 配置和状态控制

| 工具 | 功能 |
|------|------|
| `mcp__burp__set_proxy_intercept_state` | 启用/禁用 Proxy Intercept（`intercepting: true/false`） |
| `mcp__burp__set_task_execution_engine_state` | 启用/禁用 Task Execution Engine（`running: true/false`） |
| `mcp__burp__output_project_options` | 输出项目级配置（JSON） |
| `mcp__burp__set_project_options` | 设置项目级配置 |
| `mcp__burp__output_user_options` | 输出用户级配置（JSON） |
| `mcp__burp__set_user_options` | 设置用户级配置 |

```bash
# 启用代理拦截
set_proxy_intercept_state(intercepting=true)

# 禁用代理拦截
set_proxy_intercept_state(intercepting=false)

# 暂停任务执行引擎
set_task_execution_engine_state(running=false)
```

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [VNCTF2026-web-pentest-wp](../raw/web/VNCTF2026-web-pentest-wp.md) | 前端混合加密登录把 SM4 `key/iv` 经 SM2 封装给服务端；若封装值可固定复用，只需重算业务密文和 MD5 签名即可爆破。 |
