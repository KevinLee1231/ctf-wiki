---
type: family
tags: [web, family, parser-differential, ssrf, wrapper, path-traversal]
skills: [ctf-web]
raw:
  - ../raw/web/parser-wrapper-and-legacy-ssrf-tricks.md
  - ../raw/web/WMCTF2025-pdf2text-wp.md
updated: 2026-06-12
---

# Parser, Wrapper and Legacy SSRF Tricks

## 作用边界

本页是 Web 解析器差异、Wrapper 和 legacy SSRF family。它覆盖 SSRF 到 Docker/Elasticsearch/MySQL 等内部协议，URL 解析差异，PHP wrapper，Windows 8.3 短文件名，Apache/SQLite/HQL/LaTeX/PDF 解析器行为，以及库内部二段加载导致的文件读或 RCE。

共同点是：外层业务校验和内层解析器理解的资源不一致。首轮要找“哪个组件做了第二次解释”，而不是把所有案例都归成普通 SSRF 或路径穿越。

## 共同识别信号

- 目标把 URL、文件名、上传文档、SQL/HQL、PDF/SVG/Office、LaTeX、数据库连接或 wrapper 字符串交给下游库处理。
- 外层过滤看似通过，但内层组件重新解析路径、协议、host、CMap、字体、压缩数据、zip wrapper 或远程资源。
- SSRF 可访问 Docker API、metadata、Elasticsearch、MySQL、Redis、内部 HTTP 或本机端口。
- 失败状态常是外层 403/黑名单绕过成功但内层报错、返回 tar/二进制、读取空文件或连接到非预期 host。

## 最小证据

- 明确外层校验器和内层解析器分别是谁，例如 `parse_url`/curl、上传名/pdfminer、Apache ErrorDocument、HQL/H2、PHP wrapper、Docker API。
- 给出同一输入在两层的不同解释：host、path、scheme、文件名、编码、压缩层或内部资源名。
- 能用最小请求证明内层行为：端口探测、文件读、错误回显、DNS/HTTP 外带或内部 API 响应。
- 记录协议细节：HTTP method、Content-Type、是否跟随跳转、是否允许 gopher/file/zip/phar、是否需要二进制握手。

## 首轮路由

| 证据形态 | 首轮判断 | 下一跳 |
|---|---|---|
| SSRF 到 Docker API、Elasticsearch、metadata 或内部 HTTP | 先枚举协议和端点，确认是否能从文件读升级到 exec/RCE | [protocol-relay-and-internal-service-injection.md](protocol-relay-and-internal-service-injection.md), [pentest-attack-chains-and-tunneling.md](pentest-attack-chains-and-tunneling.md) |
| URL parser 和 curl/requests/后端库解释不同 | 先做 host、userinfo、IPv6、编码、重定向和 DNS rebinding 最小差异样本 | [polyglot-url-tricks-and-ssrf-leaks.md](polyglot-url-tricks-and-ssrf-leaks.md) |
| PHP `zip://`、`phar://`、`file://`、LFI wrapper | 先确认 wrapper 是否启用、路径是否可控、polyglot 是否被内层识别 | [php-lfi-ssti-ssrf-and-type-juggling.md](php-lfi-ssti-ssrf-and-type-juggling.md), [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md) |
| PDF/SVG/Office/LaTeX 解析器内部加载资源 | 先看文档内部名称和库加载路径，外层保存文件名不代表安全 | [xml-command-and-graphql-injection.md](xml-command-and-graphql-injection.md), [pdf-png-gif-and-text-stego.md](pdf-png-gif-and-text-stego.md) |
| Castor XML、XMLDecoder、pickle/gzip/CMap 等二段数据加载 | 先判断是否为反序列化 sink，再构造最小类型/数据文件触发 | [php-java-python-deserialization.md](php-java-python-deserialization.md) |
| Windows 8.3、Base64 path、SQLite/HQL 空白差异 | 先还原规范化顺序，再判断是路径穿越、SQLi 还是等值比较绕过 | [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md), [sqli-filter-and-oracle-family.md](sqli-filter-and-oracle-family.md) |
| Rogue MySQL `LOAD DATA LOCAL` 或客户端协议滥用 | 先确认目标会主动连接恶意服务，再模拟协议请求文件 | [protocol-relay-and-internal-service-injection.md](protocol-relay-and-internal-service-injection.md) |

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [WMCTF2025-pdf2text-wp](../raw/web/WMCTF2025-pdf2text-wp.md) | 上传服务只过滤保存文件名，但 pdfminer 会按 PDF 内部 Type0 字体 `/Encoding` 去拼 CMap 路径；内部名称的路径穿越比外层上传名更关键。 |

## 合并与拆分结论

本页应保留为 family。它与 `path-traversal-ssrf-upload-and-rsc.md` 不重复：后者负责 Web 业务流中的路径/上传/SSRF/RSC 链路，本页负责外层校验和内层解析器出现语义差异时的二级判断。

## 常见陷阱

- 只证明 SSRF 能访问 `127.0.0.1`，没继续枚举内部协议和升级路径。
- 外层过滤通过后就停止分析，没检查内层库实际读取的路径。
- 把 PDF/SVG/Office 当普通文件上传，忽略文档内部引用和二次加载。
- URL 绕过只看浏览器解析，没对照服务端库的真实解析。
- Rogue 协议类题没有实现握手，导致目标根本不会发送敏感数据。

## 关联技巧

- [web-first-pass-triage-and-chain-patterns.md](web-first-pass-triage-and-chain-patterns.md)
- [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md)
- [polyglot-url-tricks-and-ssrf-leaks.md](polyglot-url-tricks-and-ssrf-leaks.md)
- [protocol-relay-and-internal-service-injection.md](protocol-relay-and-internal-service-injection.md)
- [php-java-python-deserialization.md](php-java-python-deserialization.md)
- [sqli-filter-and-oracle-family.md](sqli-filter-and-oracle-family.md)
- [web-tooling.md](web-tooling.md)

## 原始资料

- [parser-wrapper-and-legacy-ssrf-tricks.md](../raw/web/parser-wrapper-and-legacy-ssrf-tricks.md)
- [WMCTF2025-pdf2text-wp](../raw/web/WMCTF2025-pdf2text-wp.md)
