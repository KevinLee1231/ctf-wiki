---
type: family
tags: [web, family, xxe, command-injection, graphql, injection]
skills: [ctf-web]
raw:
  - ../raw/web/xml-command-and-graphql-injection.md
updated: 2026-07-06
---

# XML, Command and GraphQL Injection

## 作用边界

本页是 Web 注入类二级 family，覆盖 XXE/XML 注入、命令注入、GraphQL 注入，以及少量 PHP 变量变量、可预测文件名和顺序替换绕过等“输入被二次解释”的变体。

它不是单一 technique。首轮要判断可控数据最终进入的是 XML 解析器、系统命令、GraphQL 查询构造、PHP 变量解析、文件名生成器还是过滤器重写链。不同 sink 的最小 payload、失败信号和外带方式完全不同。

## 识别信号

- 请求体、上传文档、Header、路径、条码内容、Git URL、GraphQL query 或服务端拼接字符串被后端解释。
- 响应出现文件内容、DNS/HTTP 外带、命令输出、schema 信息、错误栈、内部路径或异常过滤痕迹。
- 黑名单/正则过滤/编码转换会改变输入，危险字符串可能在多轮替换后重组。
- 需要把单点注入串到 secret、内部 API、文件读、命令执行或业务越权。

## 最小证据

- 明确 sink 类型：XML parser、shell/CLI、GraphQL resolver、PHP 变量/函数、文件名生成、正则过滤器。
- 能用一个最小请求证明输入被解释，而不是只看到报错。
- 对 blind/OOB 场景，准备可验证回连通道：DNS、HTTP、时间差、日志或服务端错误。
- 记录过滤器、编码、换行、空字节、文件格式、Content-Type 和上传后处理链。

## 首轮路由

| 证据形态 | 首轮判断 | 下一跳 |
|---|---|---|
| XML body、DOCX/Office XML、SVG、svglib、外部实体 | 先确认是否允许 DTD、外部实体、参数实体或二段文档解析，再选择 in-band/OOB XXE | [parser-wrapper-and-legacy-ssrf-tricks.md](parser-wrapper-and-legacy-ssrf-tricks.md), [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md) |
| Header、日志、`X-Forwarded-For` 被拼进 XML 或模板 | 先找日志/模板/后端协议拼接点，不要只测 body 参数 | [auth-bypass-cookies-and-hidden-routes.md](auth-bypass-cookies-and-hidden-routes.md) |
| `;`, newline, backtick, `$()`, sendmail 参数、Git CLI path、条码内容进入命令 | 先确定 shell/CLI 参数边界和换行处理，再用最短命令副作用验证 | [sqli-upload-deser-and-command-rce.md](sqli-upload-deser-and-command-rce.md), [ruby-php-upload-and-ssti-rce.md](ruby-php-upload-and-ssti-rce.md) |
| GraphQL introspection、batching、alias、resolver 字符串插值 | 先枚举 schema 和 mutation，再判断是越权、批量绕限还是后端注入 | [auth-edge-cases-and-protocol-bypasses.md](auth-edge-cases-and-protocol-bypasses.md), [workflow-runner-internal-api-chain.md](workflow-runner-internal-api-chain.md) |
| PHP `$$var`、`uniqid()`、时间种子文件名 | 先确认变量名/时间/随机源可控，再转成文件读、session、上传或路径猜测 | [php-lfi-ssti-ssrf-and-type-juggling.md](php-lfi-ssti-ssrf-and-type-juggling.md), [auth-bypass-cookies-and-hidden-routes.md](auth-bypass-cookies-and-hidden-routes.md) |
| 顺序正则替换、黑名单多轮清洗 | 先本地复现过滤器，构造“删除后重组”的最小 payload | [xss-dom-and-browser-tricks.md](xss-dom-and-browser-tricks.md), [parser-wrapper-and-legacy-ssrf-tricks.md](parser-wrapper-and-legacy-ssrf-tricks.md) |

## 合并与拆分结论

本页应保留为 family。XXE、命令注入和 GraphQL 注入的 sink 完全不同，不应伪装成单个 technique；但它们共享“输入被服务端二次解释”的首轮判断，放在一个二级入口可以减少在 Web 首轮页中堆细节。

当前不拆出 `xxe.md`、`command-injection.md`、`graphql-injection.md`：raw 多为短案例集合，且已有其它 family 承接后续链路。后续若某一类积累多篇 WP，再拆成具体 technique。

## 常见陷阱

- 看到 XML 就只测 body，忽略上传容器、Office/SVG 内部 XML 和 Header 拼接。
- 命令注入只试分号，没验证换行、参数注入、CLI 子命令和环境差异。
- GraphQL 只跑 introspection，不继续找 mutation、batching、alias 和 resolver 拼接。
- 黑名单绕过只试编码，不在本地复现多轮替换顺序。
- blind/OOB 注入没有准备回连证据，导致成功和失败不可区分。

## 关联技巧

- [web-first-pass-triage-and-chain-patterns.md](web-first-pass-triage-and-chain-patterns.md)
- [parser-wrapper-and-legacy-ssrf-tricks.md](parser-wrapper-and-legacy-ssrf-tricks.md)
- [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md)
- [sqli-upload-deser-and-command-rce.md](sqli-upload-deser-and-command-rce.md)
- [ruby-php-upload-and-ssti-rce.md](ruby-php-upload-and-ssti-rce.md)
- [auth-edge-cases-and-protocol-bypasses.md](auth-edge-cases-and-protocol-bypasses.md)
- [web-tooling.md](web-tooling.md)

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [SUCTF2026-jdbc-masterWP](../raw/web/SUCTF2026-jdbc-masterWP.md) | Unicode 长 `ſ` 绕过 `/suctf` 路径过滤后，可控 JDBC driver/URL 走 Kingbase `ConfigurePath` 与 Spring XML beans 无外连加载。 |

## 原始资料

- [xml-command-and-graphql-injection.md](../raw/web/xml-command-and-graphql-injection.md)
