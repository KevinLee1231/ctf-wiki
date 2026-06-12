---
type: family
tags: [web, family, path-traversal, ssrf, upload, file-read, internal-service]
skills: [ctf-web]
raw:
  - ../raw/web/path-traversal-ssrf-upload-and-rsc.md
updated: 2026-06-11
---

# Path Traversal, SSRF, Upload and RSC

## 作用边界

本页是文件访问和内部服务链 family。共同模式不是某个单点 payload，而是 Web 服务把用户可控路径、URL、压缩包、渲染器输入或内部协议交给文件系统、网络客户端、解包器、PDF/SVG/Office 解析器、RSC 反序列化或后端消息系统处理。

如果题目的核心已经是单纯 SQL oracle、JWT、Node 原型污染或 Java 反序列化，应转到对应 technique；本页只负责从“可控资源定位”分流到读取、SSRF、上传落地或内部执行链。

## 变体路由

| 信号 | 最小证据 | 下一跳 |
|---|---|---|
| `../`、编码斜杠、Unicode 点、`os.path.join`、Nginx alias | 能读到源码、`.env`、配置、备份、`.git/.bzr` 或 `/proc` 信息 | [php-lfi-ssti-ssrf-and-type-juggling.md](php-lfi-ssti-ssrf-and-type-juggling.md)、[polyglot-url-tricks-and-ssrf-leaks.md](polyglot-url-tricks-and-ssrf-leaks.md) |
| ZIP symlink、ZipSlip、解压覆盖、自更新包 | 上传包中的路径或软链接能越过目标目录 | [ruby-php-upload-and-ssti-rce.md](ruby-php-upload-and-ssti-rce.md)、[artifact-trust-ssrf-to-node-require-rce.md](artifact-trust-ssrf-to-node-require-rce.md) |
| ExifTool、Image/PDF/SVG/Office 渲染、WeasyPrint、CairoSVG | 解析器会发起本地文件读取、外部请求或命令执行 | [xml-command-and-graphql-injection.md](xml-command-and-graphql-injection.md)、[known-cves-and-n-day-exploits.md](known-cves-and-n-day-exploits.md) |
| SSRF 到 SMTP/MySQL/Gopher/metadata/internal API | 能控制协议、Host、跳转、body 或 gopher payload，且响应/时间可观测 | [artifact-trust-ssrf-to-node-require-rce.md](artifact-trust-ssrf-to-node-require-rce.md)、[parser-wrapper-and-legacy-ssrf-tricks.md](parser-wrapper-and-legacy-ssrf-tricks.md) |
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

## 原始资料

- [path-traversal-ssrf-upload-and-rsc.md](../raw/web/path-traversal-ssrf-upload-and-rsc.md)
