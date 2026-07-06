---
type: family
tags: [web, family, path-traversal, ssrf, upload, file-read, internal-service]
skills: [ctf-web]
raw:
  - ../raw/web/path-traversal-ssrf-upload-and-rsc.md
  - ../raw/web/WMCTF2025-pdf2text-wp.md
  - ../raw/web/WMCTF2025-rustdesk-change-client-backend-wp.md
  - ../raw/web/D3CTF2019-ezupload-wp.md
updated: 2026-07-06
---

# Path Traversal, SSRF, Upload and RSC

## 作用边界

本页是文件访问和内部服务链 family。共同模式不是某个单点 payload，而是 Web 服务把用户可控路径、URL、压缩包、渲染器输入或内部协议交给文件系统、网络客户端、解包器、PDF/SVG/Office 解析器、RSC 反序列化或后端消息系统处理。

如果题目的核心已经是单纯 SQL oracle、JWT、Node 原型污染或 Java 反序列化，应转到对应 technique；本页只负责从“可控资源定位”分流到读取、SSRF、上传落地或内部执行链。

## 识别信号

- 用户输入会成为路径、URL、文件名、归档成员名、渲染资源、redirect 目标、组件 payload 或内部消息字段。
- 后端会替用户读取文件、下载 URL、解包归档、渲染 PDF/SVG/Office、执行 server action 或连接内部服务。
- 响应、错误、时间、导出文件、截图或二段请求能证明资源定位已经进入后端解释层。
- 过滤逻辑只检查字符串表面形式，没有绑定最终解析后的路径、host、协议、软链接或解包落点。

## 最小证据

- 保存一个正常资源请求和一个最小异常请求，标出可控字段如何变成路径、URL、归档成员名、渲染资源或内部消息字段。
- 记录后端最终解析结果：归一化路径、真实 host/IP、协议、解包落点、软链接目标、渲染器资源名或 server action。
- 文件读路线至少证明能读取源码、配置、`.env`、`.git`、上传目录、`/proc` 或其它稳定敏感资源。
- SSRF/内部服务路线至少证明目标服务、协议细节、响应/时间/DNS 反馈和是否可升级到内部 API、文件写入或 RCE。
- 上传/解包路线至少确认文件内容可控、落点可达、解析规则或 Web 访问路径，不只看扩展名是否放行。

## 变体路由

| 信号 | 最小证据 | 下一跳 |
|---|---|---|
| `../`、编码斜杠、Unicode 点、`os.path.join`、Nginx alias | 能读到源码、`.env`、配置、备份、`.git/.bzr` 或 `/proc` 信息 | [php-lfi-ssti-ssrf-and-type-juggling.md](php-lfi-ssti-ssrf-and-type-juggling.md)、[polyglot-url-tricks-and-ssrf-leaks.md](polyglot-url-tricks-and-ssrf-leaks.md) |
| ZIP symlink、ZipSlip、解压覆盖、自更新包 | 上传包中的路径或软链接能越过目标目录 | [ruby-php-upload-and-ssti-rce.md](ruby-php-upload-and-ssti-rce.md)、[artifact-trust-ssrf-to-node-require-rce.md](artifact-trust-ssrf-to-node-require-rce.md) |
| ExifTool、Image/PDF/SVG/Office 渲染、WeasyPrint、CairoSVG | 解析器会发起本地文件读取、外部请求或命令执行 | [xml-command-and-graphql-injection.md](xml-command-and-graphql-injection.md)、[known-cves-and-n-day-exploits.md](known-cves-and-n-day-exploits.md) |
| SSRF 到 SMTP/MySQL/Gopher/metadata/internal API | 能控制协议、Host、跳转、body 或 gopher payload，且响应/时间可观测 | [artifact-trust-ssrf-to-node-require-rce.md](artifact-trust-ssrf-to-node-require-rce.md)、[parser-wrapper-and-legacy-ssrf-tricks.md](parser-wrapper-and-legacy-ssrf-tricks.md) |
| 远控/relay/客户端 backend 让目标主动连接内网服务 | 能控制 relay/server 地址和明文字段，目标会连接本机 Redis/Docker/API | [protocol-relay-and-internal-service-injection.md](protocol-relay-and-internal-service-injection.md) |
| React Server Components Flight、服务端组件反序列化 | 响应头/Flight payload 表明 RSC 解析用户输入，且能触发 server action 或 redirect | [node-and-prototype.md](node-and-prototype.md)、[workflow-runner-internal-api-chain.md](workflow-runner-internal-api-chain.md) |
| AMQP/TLS、内部 broker、MITM/重写 | Web 服务与内部消息系统之间存在可拦截或可重放的身份字段 | [workflow-runner-internal-api-chain.md](workflow-runner-internal-api-chain.md) |

## 合并与拆分结论

- 保留为 family：raw 覆盖路径穿越、SSRF、上传、渲染器文件读、RSC 和内部协议链，它们共享“资源定位交给后端解释”的首轮判断。
- 不合并进 PHP/LFI family：本页不限 PHP，也不限本地文件包含；它覆盖跨语言和跨组件的文件/网络/解包链。
- 暂不拆 RSC 独立页：当前 RSC 内容更像 Node/Web 内部执行链的一个变体，先通过本页和 [node-and-prototype.md](node-and-prototype.md) 入链。

## 常见误判

- 只证明能 SSRF 外网回连，却没有证明内网目标、协议和响应通道。
- 上传链只看扩展名，不看解压路径、软链接、MIME、服务器映射和访问路径。
- 路径穿越绕过依赖编码层时，代理、框架、Web 服务器和文件系统可能各解码一次，必须逐层确认。
- RSC/Flight 题不要只按 XSS 处理；它的危险点常在服务端组件反序列化和 server action。

## 关联页面

- [web-first-pass-triage-and-chain-patterns.md](web-first-pass-triage-and-chain-patterns.md)
- [php-lfi-ssti-ssrf-and-type-juggling.md](php-lfi-ssti-ssrf-and-type-juggling.md)
- [ruby-php-upload-and-ssti-rce.md](ruby-php-upload-and-ssti-rce.md)
- [artifact-trust-ssrf-to-node-require-rce.md](artifact-trust-ssrf-to-node-require-rce.md)
- [path-confusion-to-signed-internal-request-chain.md](path-confusion-to-signed-internal-request-chain.md)
- [polyglot-url-tricks-and-ssrf-leaks.md](polyglot-url-tricks-and-ssrf-leaks.md)
- [protocol-relay-and-internal-service-injection.md](protocol-relay-and-internal-service-injection.md)

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [WMCTF2025-pdf2text-wp](../raw/web/WMCTF2025-pdf2text-wp.md) | 外层文件名过滤不含 `..` 不够，PDF 内部 `/Encoding` 名称仍可路径穿越到上传目录并触发解析器内部资源加载。 |
| [WMCTF2025-rustdesk-change-client-backend-wp](../raw/web/WMCTF2025-rustdesk-change-client-backend-wp.md) | 目标不是传统 HTTP SSRF，而是 RustDesk 受控端按 relay 消息主动连接 `127.0.0.1:6379`，再让可控 uuid 字段成为 Redis 协议内容。 |
| [D3CTF2019-ezupload-wp](../raw/web/D3CTF2019-ezupload-wp.md) | 反序列化给出写文件能力后，真正瓶颈是用 `glob://` 爆破随机上传目录绝对路径，并用 `.htaccess` 改解析规则把文本文件推进到 PHP 执行。 |
| [D3CTF2023-d3readfile-wp](../raw/misc/D3CTF2023-d3readfile-wp.md) | 任意文件读不知道 flag 路径时，先读取 locate 数据库并在本地搜索目标路径。 |
| [ACTF2026-real-dlsite-wp](../raw/web/ACTF2026-real-dlsite-wp.md) | go-drive `CleanPath`/`Join` 不一致先拿 app RCE，再串 PHP open_basedir check/use race 和 StorageBox setuid TOCTOU 读 root flag。 |
| [D3CTF2023-d3cloud-wp](../raw/web/D3CTF2023-d3cloud-wp.md) | laravel-admin 上传 zip 后拼接 unzip 命令，先控制文件名进入命令执行。 |
| [D3CTF2023-d3go-wp](../raw/web/D3CTF2023-d3go-wp.md) | Go embed/static 目录穿越拿源码，再用 Gorm 软删除注入和 zip 解压覆盖 self-update。 |
| [D3CTF2025-d3jtar-wp](../raw/web/D3CTF2025-d3jtar-wp.md) | jtar 文件名 Unicode 到 ASCII 转换造成后缀变化，先借备份/恢复把上传文件变成 JSP。 |
| [HGAME2026-babyweb-wp](../raw/web/HGAME2026-babyweb-wp.md) | PHP 上传允许 .php/.htaccess 并可落 WebShell，先证明可控文件内容、解析规则和内网 pivot。 |
| [HGAME2026-easyuu-wp](../raw/web/HGAME2026-easyuu-wp.md) | 可控目录列表找到更新包源码，上传后门 `path1` 可覆盖 `./update/easyuu`，再借自更新机制执行自定义程序。 |
| [NCTF2026-n-minsite-wp](../raw/web/NCTF2026-n-minsite-wp.md) | MaxSite 源码可由 `/?key` 获取，后台上传入口只校验登录态；上传 HTML/JS 后诱导 bot 读取后台 flag。 |
| [RCTF2025-514s-heart-wp](../raw/web/RCTF2025-514s-heart-wp.md) | Koishi 插件静态路由目录穿越先读配置拿后台密码，再用配置 `${{ ... }}` 表达式执行命令。 |
| [SU_wmsWP](../raw/web/SU_wmsWP.md) | JEECG `/rest/*` 白名单放行且 Spring `params` 可由 POST body 命中后台方法；模板 ZIP 解压目录穿越写 JSP。 |
| [VNCTF2026-markdown2world-wp](../raw/web/VNCTF2026-markdown2world-wp.md) | Pandoc 转 docx 时会读取 Markdown 图片目标并嵌入 `word/media/`；构造本地资源引用后解压 docx 取回文件内容。 |

## 原始资料

- [path-traversal-ssrf-upload-and-rsc.md](../raw/web/path-traversal-ssrf-upload-and-rsc.md)
- [WMCTF2025-pdf2text-wp](../raw/web/WMCTF2025-pdf2text-wp.md)
- [WMCTF2025-rustdesk-change-client-backend-wp](../raw/web/WMCTF2025-rustdesk-change-client-backend-wp.md)
- [D3CTF2019-ezupload-wp](../raw/web/D3CTF2019-ezupload-wp.md)
