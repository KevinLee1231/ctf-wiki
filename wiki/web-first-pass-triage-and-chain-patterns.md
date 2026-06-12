---
type: family
tags: [web, family, triage]
skills: [ctf-web]
raw:
  - ../raw/web/web-first-pass-triage-and-chain-patterns.md
updated: 2026-06-11
---

# First-Pass Triage and Chain Patterns

## 作用边界

本页是 Web 方向首轮 family。它只负责把题面、源码、请求行为和初始回显分流到更具体的 family / technique；不承载具体 payload 字典，也不替代 raw WP。

首轮目标是回答四个问题：

1. 输入从哪里进入：URL、Header、Cookie、body、文件、模板、浏览器、内部 API 还是第三方协议。
2. 哪一层先解释它：代理、框架路由、认证中间件、解析器、模板引擎、数据库、浏览器或内部 worker。
3. 结果能观察到什么：回显、错误、状态码、时间、跳转、文件、token、bot 行为或二段请求。
4. 最短链路是什么：读源码/secret、绕认证、访问内部服务、执行代码、浏览器外带或业务状态竞争。

## 首轮路由

| 初始信号 | 先去哪里 | 判断重点 |
|---|---|---|
| 登录态、cookie、session、hidden route、IDOR、代理 ACL | [auth-bypass-cookies-and-hidden-routes.md](auth-bypass-cookies-and-hidden-routes.md) | 服务端到底以哪个身份字段、路由视图或信任边界授权。 |
| JWT、HMAC、重复 JSON key、签名字段和 parser 差异 | [auth-jwt.md](auth-jwt.md)、[json-duplicate-key-hmac-parser-differential.md](json-duplicate-key-hmac-parser-differential.md) | 签名层和业务层看到的对象是否一致。 |
| SQLi、NoSQL、过滤绕过、盲注、二阶触发 | [sqli-filter-and-oracle-family.md](sqli-filter-and-oracle-family.md) | 回显、错误、时间、长度和持久化触发点。 |
| PHP/LFI/SSTI/SSRF/XXE/type juggling | [php-lfi-ssti-ssrf-and-type-juggling.md](php-lfi-ssti-ssrf-and-type-juggling.md) | 服务端解释层差异是否通向读文件、内部访问、认证绕过或 RCE。 |
| 路径穿越、上传、ZIP/软链接、渲染器、RSC、内部协议 | [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md) | 可控资源定位是否交给文件系统、网络客户端、解包器或内部服务。 |
| 语言 eval、模板 eval、WebShell、反序列化、命令拼接 | [ruby-php-upload-and-ssti-rce.md](ruby-php-upload-and-ssti-rce.md)、[php-java-python-deserialization.md](php-java-python-deserialization.md) | 输入是否已经进入可执行解释层，gadget 或执行点是否可达。 |
| Node、prototype pollution、workflow runner、internal API | [node-and-prototype.md](node-and-prototype.md)、[workflow-runner-internal-api-chain.md](workflow-runner-internal-api-chain.md) | 对象合并、包生命周期、worker/runner 信任链和内部 API 权限。 |
| XSS、DOM、admin bot、CSP、XS-Leak、cache/MIME | [xss-dom-and-browser-tricks.md](xss-dom-and-browser-tricks.md)、[csp-xsleak-and-browser-exfiltration.md](csp-xsleak-and-browser-exfiltration.md) | 浏览器上下文、CSP/同源限制、外带 oracle 和 bot 触发条件。 |
| 已知组件、CVE、WordPress/Chromium/ExifTool/框架版本 | [known-cves-and-n-day-exploits.md](known-cves-and-n-day-exploits.md) | 版本、补丁边界、利用前置条件和是否需要串其它漏洞。 |
| Jail、表达式沙箱、Python/Web 混合过滤 | [pyjails.md](pyjails.md) | 过滤发生在语言、表达式、对象图还是 Web 包装层。 |

## 失败后 pivot

- 一个请求没有差异：固定正常业务流，逐个改变 method、path、header、cookie、body、content-type 和编码层。
- 有错误但不能外带：优先找源码、配置、secret、日志、debug endpoint 或内部回显，不要急着堆 RCE payload。
- 有 SSRF/内部访问但无响应：转 blind oracle，观察时间、DNS、缓存、文件写入、队列状态或二段请求。
- 有 XSS 但拿不到数据：转 CSP/XS-Leak，构造可重复的浏览器 oracle。
- 已经 RCE 但权限低：回到文件、环境变量、容器/云 metadata、内部服务和凭据复用。

## 关联页面

- [web-tooling.md](web-tooling.md)
- [auth-bypass-cookies-and-hidden-routes.md](auth-bypass-cookies-and-hidden-routes.md)
- [php-lfi-ssti-ssrf-and-type-juggling.md](php-lfi-ssti-ssrf-and-type-juggling.md)
- [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md)
- [ruby-php-upload-and-ssti-rce.md](ruby-php-upload-and-ssti-rce.md)
- [xss-dom-and-browser-tricks.md](xss-dom-and-browser-tricks.md)
- [sqli-filter-and-oracle-family.md](sqli-filter-and-oracle-family.md)

## WP 案例沉淀

本节只抽取 raw WP 中可复用的识别信号和下一跳，不替代原始题解正文。

| Raw WP | 复用信号 | 下一跳 |
|---|---|---|
| [ACTF2026-12307-wp](../raw/web/ACTF2026-12307-wp.md) | 认证、会话或 token 语义异常，先比较签名层、业务层和身份绑定。 | [auth-edge-cases-and-protocol-bypasses.md](auth-edge-cases-and-protocol-bypasses.md) |
| [ACTF2026-aaa26-wp](../raw/web/ACTF2026-aaa26-wp.md) | 认证、会话或 token 语义异常，先比较签名层、业务层和身份绑定。 | [auth-edge-cases-and-protocol-bypasses.md](auth-edge-cases-and-protocol-bypasses.md) |
| [ACTF2026-gomysql-wp](../raw/web/ACTF2026-gomysql-wp.md) | SQL/数据库注入或过滤 oracle，先比较回显、错误、时间、长度和二阶触发点。 | [sqli-filter-and-oracle-family.md](sqli-filter-and-oracle-family.md) |
| [ACTF2026-real-dlsite-wp](../raw/web/ACTF2026-real-dlsite-wp.md) | 文件、路径、模板、反序列化或上传链，先证明可控路径/内容再串读取或 RCE。 | [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md) |
| [ACTF2026-tinj-wp](../raw/web/ACTF2026-tinj-wp.md) | JPHP/Jetty 字符字节转换绕过路由后进入 eval、JAR/classloader 和内存马链，先确认运行时边界。 | [php-lfi-ssti-ssrf-and-type-juggling.md](php-lfi-ssti-ssrf-and-type-juggling.md) |
| [Bugku-API-2-The-SeQueL-wp](../raw/web/Bugku-API-2-The-SeQueL-wp.md) | SQL/数据库注入或过滤 oracle，先比较回显、错误、时间、长度和二阶触发点。 | [sqli-filter-and-oracle-family.md](sqli-filter-and-oracle-family.md) |
| [Bugku-Serial-Killer-wp](../raw/web/Bugku-Serial-Killer-wp.md) | Web 首轮信号不唯一，先固定正常业务流和最小请求差异，再 pivot 到具体漏洞页。 | [web-first-pass-triage-and-chain-patterns.md](web-first-pass-triage-and-chain-patterns.md) |
| [HGAME2026-博丽神社的绘马挂-wp](../raw/web/HGAME2026-博丽神社的绘马挂-wp.md) | Web 首轮信号不唯一，先固定正常业务流和最小请求差异，再 pivot 到具体漏洞页。 | [web-first-pass-triage-and-chain-patterns.md](web-first-pass-triage-and-chain-patterns.md) |
| [HGAME2026-魔理沙的魔法目录-wp](../raw/web/HGAME2026-魔理沙的魔法目录-wp.md) | Web 首轮信号不唯一，先固定正常业务流和最小请求差异，再 pivot 到具体漏洞页。 | [web-first-pass-triage-and-chain-patterns.md](web-first-pass-triage-and-chain-patterns.md) |
| [HGAME2026-文文新闻-wp](../raw/web/HGAME2026-文文新闻-wp.md) | Web 首轮信号不唯一，先固定正常业务流和最小请求差异，再 pivot 到具体漏洞页。 | [web-first-pass-triage-and-chain-patterns.md](web-first-pass-triage-and-chain-patterns.md) |
| [HGAME2026-babyweb-wp](../raw/web/HGAME2026-babyweb-wp.md) | PHP 上传允许 .php/.htaccess 并可落 WebShell，先证明可控文件内容、解析规则和内网 pivot。 | [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md) |
| [HGAME2026-easyuu-wp](../raw/web/HGAME2026-easyuu-wp.md) | Web 首轮信号不唯一，先固定正常业务流和最小请求差异，再 pivot 到具体漏洞页。 | [web-first-pass-triage-and-chain-patterns.md](web-first-pass-triage-and-chain-patterns.md) |
| [HGAME2026-ezcc-wp](../raw/web/HGAME2026-ezcc-wp.md) | 文件、路径、模板、反序列化或上传链，先证明可控路径/内容再串读取或 RCE。 | [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md) |
| [HGAME2026-my-little-assistant-wp](../raw/web/HGAME2026-my-little-assistant-wp.md) | SSRF/内部服务/可信 artifact 链，先固定内部 URL、secret 流向和二段利用边界。 | [artifact-trust-ssrf-to-node-require-rce.md](artifact-trust-ssrf-to-node-require-rce.md) |
| [HGAME2026-my-monitor-wp](../raw/web/HGAME2026-my-monitor-wp.md) | SSRF/内部服务/可信 artifact 链，先固定内部 URL、secret 流向和二段利用边界。 | [artifact-trust-ssrf-to-node-require-rce.md](artifact-trust-ssrf-to-node-require-rce.md) |
| [HGAME2026-vidarshop-wp](../raw/web/HGAME2026-vidarshop-wp.md) | Web 首轮信号不唯一，先固定正常业务流和最小请求差异，再 pivot 到具体漏洞页。 | [web-first-pass-triage-and-chain-patterns.md](web-first-pass-triage-and-chain-patterns.md) |
| [LilacCTF2026-checkin-wp](../raw/web/LilacCTF2026-checkin-wp.md) | Web 首轮信号不唯一，先固定正常业务流和最小请求差异，再 pivot 到具体漏洞页。 | [web-first-pass-triage-and-chain-patterns.md](web-first-pass-triage-and-chain-patterns.md) |
| [LilacCTF2026-nailong-wp](../raw/web/LilacCTF2026-nailong-wp.md) | Pickle/PyTorch 模型加载执行链和扫描绕过，先确认反序列化入口、ZIP/CRC 信任边界和命令执行点。 | [php-java-python-deserialization.md](php-java-python-deserialization.md) |
| [LilacCTF2026-playground-wp](../raw/web/LilacCTF2026-playground-wp.md) | 浏览器执行模型或 bot 行为可控，先确认 sink、CSP、触发上下文和 exfil 通道。 | [xss-dom-and-browser-tricks.md](xss-dom-and-browser-tricks.md) |
| [LilacCTF2026-safe-sql-wp](../raw/web/LilacCTF2026-safe-sql-wp.md) | SQL/数据库注入或过滤 oracle，先比较回显、错误、时间、长度和二阶触发点。 | [sqli-filter-and-oracle-family.md](sqli-filter-and-oracle-family.md) |
| [NCTF2026-n-minsite-wp](../raw/web/NCTF2026-n-minsite-wp.md) | Web 首轮信号不唯一，先固定正常业务流和最小请求差异，再 pivot 到具体漏洞页。 | [web-first-pass-triage-and-chain-patterns.md](web-first-pass-triage-and-chain-patterns.md) |
| [NCTF2026-n-rustpica-wp](../raw/web/NCTF2026-n-rustpica-wp.md) | 文件、路径、模板、反序列化或上传链，先证明可控路径/内容再串读取或 RCE。 | [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md) |
| [NCTF2026-openshell-wp](../raw/web/NCTF2026-openshell-wp.md) | 文件、路径、模板、反序列化或上传链，先证明可控路径/内容再串读取或 RCE。 | [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md) |
| [RCTF2025-514s-heart-wp](../raw/web/RCTF2025-514s-heart-wp.md) | Web 首轮信号不唯一，先固定正常业务流和最小请求差异，再 pivot 到具体漏洞页。 | [web-first-pass-triage-and-chain-patterns.md](web-first-pass-triage-and-chain-patterns.md) |
| [RCTF2025-auth-wp](../raw/web/RCTF2025-auth-wp.md) | 认证、会话或 token 语义异常，先比较签名层、业务层和身份绑定。 | [auth-edge-cases-and-protocol-bypasses.md](auth-edge-cases-and-protocol-bypasses.md) |
| [RCTF2025-author-plus-wp](../raw/web/RCTF2025-author-plus-wp.md) | 认证、会话或 token 语义异常，先比较签名层、业务层和身份绑定。 | [auth-edge-cases-and-protocol-bypasses.md](auth-edge-cases-and-protocol-bypasses.md) |
| [RCTF2025-author-wp](../raw/web/RCTF2025-author-wp.md) | 认证、会话或 token 语义异常，先比较签名层、业务层和身份绑定。 | [auth-edge-cases-and-protocol-bypasses.md](auth-edge-cases-and-protocol-bypasses.md) |
| [RCTF2025-maybe-easy-wp](../raw/web/RCTF2025-maybe-easy-wp.md) | Web 首轮信号不唯一，先固定正常业务流和最小请求差异，再 pivot 到具体漏洞页。 | [web-first-pass-triage-and-chain-patterns.md](web-first-pass-triage-and-chain-patterns.md) |
| [RCTF2025-photographer-wp](../raw/web/RCTF2025-photographer-wp.md) | 文件、路径、模板、反序列化或上传链，先证明可控路径/内容再串读取或 RCE。 | [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md) |
| [RCTF2025-rootkb-dash-wp](../raw/web/RCTF2025-rootkb-dash-wp.md) | SSRF/内部服务/可信 artifact 链，先固定内部 URL、secret 流向和二段利用边界。 | [artifact-trust-ssrf-to-node-require-rce.md](artifact-trust-ssrf-to-node-require-rce.md) |
| [RCTF2025-rootkb-wp](../raw/web/RCTF2025-rootkb-wp.md) | SSRF/内部服务/可信 artifact 链，先固定内部 URL、secret 流向和二段利用边界。 | [artifact-trust-ssrf-to-node-require-rce.md](artifact-trust-ssrf-to-node-require-rce.md) |
| [RCTF2025-ultimatefreeloader-wp](../raw/web/RCTF2025-ultimatefreeloader-wp.md) | 下单和退款使用不同 Redis lock key，先并发触发购买/退款窗口并验证余额、优惠券和商品状态不互斥。 | [race-condition-and-concurrency-exploits.md](race-condition-and-concurrency-exploits.md) |
| [Spirit2026-5-artifact-relay-wp](../raw/web/Spirit2026-5-artifact-relay-wp.md) | SSRF/内部服务/可信 artifact 链，先固定内部 URL、secret 流向和二段利用边界。 | [artifact-trust-ssrf-to-node-require-rce.md](artifact-trust-ssrf-to-node-require-rce.md) |
| [Spirit2026-5-flowforge-wp](../raw/web/Spirit2026-5-flowforge-wp.md) | SSRF/内部服务/可信 artifact 链，先固定内部 URL、secret 流向和二段利用边界。 | [artifact-trust-ssrf-to-node-require-rce.md](artifact-trust-ssrf-to-node-require-rce.md) |
| [Spirit2026-5-trust-collapse-wp](../raw/web/Spirit2026-5-trust-collapse-wp.md) | SSRF/内部服务/可信 artifact 链，先固定内部 URL、secret 流向和二段利用边界。 | [artifact-trust-ssrf-to-node-require-rce.md](artifact-trust-ssrf-to-node-require-rce.md) |
| [SU_cmsAgainWP](../raw/web/SU_cmsAgainWP.md) | 文件、路径、模板、反序列化或上传链，先证明可控路径/内容再串读取或 RCE。 | [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md) |
| [SU_jdbc-masterWP](../raw/web/SU_jdbc-masterWP.md) | SQL/数据库注入或过滤 oracle，先比较回显、错误、时间、长度和二阶触发点。 | [sqli-filter-and-oracle-family.md](sqli-filter-and-oracle-family.md) |
| [SU_Note_revWP](../raw/web/SU_Note_revWP.md) | 认证、会话或 token 语义异常，先比较签名层、业务层和身份绑定。 | [auth-edge-cases-and-protocol-bypasses.md](auth-edge-cases-and-protocol-bypasses.md) |
| [SU_NoteWP](../raw/web/SU_NoteWP.md) | 认证、会话或 token 语义异常，先比较签名层、业务层和身份绑定。 | [auth-edge-cases-and-protocol-bypasses.md](auth-edge-cases-and-protocol-bypasses.md) |
| [SU_sqliWP](../raw/web/SU_sqliWP.md) | SQL/数据库注入或过滤 oracle，先比较回显、错误、时间、长度和二阶触发点。 | [sqli-filter-and-oracle-family.md](sqli-filter-and-oracle-family.md) |
| [SU_uriWP](../raw/web/SU_uriWP.md) | 文件、路径、模板、反序列化或上传链，先证明可控路径/内容再串读取或 RCE。 | [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md) |
| [SU_wmsWP](../raw/web/SU_wmsWP.md) | 文件、路径、模板、反序列化或上传链，先证明可控路径/内容再串读取或 RCE。 | [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md) |
| [VNCTF2026-black-coffee-wp](../raw/web/VNCTF2026-black-coffee-wp.md) | Java 原生反序列化 gadget 链，先确认编码层、触发类、TemplatesImpl/POJONode 等可用 gadget。 | [php-java-python-deserialization.md](php-java-python-deserialization.md) |
| [VNCTF2026-i-really-really-really-and-revenge-wp](../raw/web/VNCTF2026-i-really-really-really-and-revenge-wp.md) | Web 首轮信号不唯一，先固定正常业务流和最小请求差异，再 pivot 到具体漏洞页。 | [web-first-pass-triage-and-chain-patterns.md](web-first-pass-triage-and-chain-patterns.md) |
| [VNCTF2026-i-really-really-really-ultimate-wp](../raw/web/VNCTF2026-i-really-really-really-ultimate-wp.md) | Web 首轮信号不唯一，先固定正常业务流和最小请求差异，再 pivot 到具体漏洞页。 | [web-first-pass-triage-and-chain-patterns.md](web-first-pass-triage-and-chain-patterns.md) |
| [VNCTF2026-markdown2world-wp](../raw/web/VNCTF2026-markdown2world-wp.md) | 文件、路径、模板、反序列化或上传链，先证明可控路径/内容再串读取或 RCE。 | [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md) |
| [VNCTF2026-more-black-coffee-wp](../raw/web/VNCTF2026-more-black-coffee-wp.md) | Java 反序列化继续加载内存马，先确认 FilterShell 注入点、触发链和容器运行时边界。 | [php-java-python-deserialization.md](php-java-python-deserialization.md) |
| [VNCTF2026-signin-wp](../raw/web/VNCTF2026-signin-wp.md) | PHP `include` 可控且过滤 `php://filter` 常见写法，先用短 `data:,<?=...` 伪协议构造执行内容。 | [php-lfi-ssti-ssrf-and-type-juggling.md](php-lfi-ssti-ssrf-and-type-juggling.md) |
| [VNCTF2026-web-pentest-wp](../raw/web/VNCTF2026-web-pentest-wp.md) | Web 首轮信号不唯一，先固定正常业务流和最小请求差异，再 pivot 到具体漏洞页。 | [web-first-pass-triage-and-chain-patterns.md](web-first-pass-triage-and-chain-patterns.md) |
| [D3CTF2019-babyxss-wp](../raw/web/D3CTF2019-babyxss-wp.md) | XSS/CSP/browser bot 是主线，先确认 sink、CSP 约束和可用 exfil 通道。 | [xss-dom-and-browser-tricks.md](xss-dom-and-browser-tricks.md) |
| [D3CTF2019-d3guestbook-wp](../raw/web/D3CTF2019-d3guestbook-wp.md) | HTML 白名单、JSONP callback 和 CSRF token/sessionid 绑定组合，先构造表单劫持拿管理员 token。 | [xss-dom-and-browser-tricks.md](xss-dom-and-browser-tricks.md) |
| [D3CTF2019-easyweb-wp](../raw/web/D3CTF2019-easyweb-wp.md) | 二次 SQL 注入控制 Smarty 模板，再经 php:phar 触发反序列化 POP 链。 | [sqli-upload-deser-and-command-rce.md](sqli-upload-deser-and-command-rce.md) |
| [D3CTF2019-ezts-wp](../raw/web/D3CTF2019-ezts-wp.md) | Sequelize JSON path 注入进入后台后，再用 lodash.defaultsDeep 原型污染 EJS RCE。 | [node-and-prototype.md](node-and-prototype.md) |
| [D3CTF2019-ezupload-wp](../raw/web/D3CTF2019-ezupload-wp.md) | 反序列化任意文件写结合 glob 爆破真实上传路径和 .htaccess 解析绕过。 | [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md) |
| [D3CTF2019-fake-onelinephp-wp](../raw/web/D3CTF2019-fake-onelinephp-wp.md) | Windows PHP 文件包含可走 WebDAV/SMB 远程包含，再从 git 和内网凭据继续 pivot。 | [php-lfi-ssti-ssrf-and-type-juggling.md](php-lfi-ssti-ssrf-and-type-juggling.md) |
| [D3CTF2019-showhub-wp](../raw/web/D3CTF2019-showhub-wp.md) | insert 注入可用 on duplicate key update 改管理员密码，后续还需 request smuggling 绕过内网 IP 限制。 | [sqli-upload-deser-and-command-rce.md](sqli-upload-deser-and-command-rce.md) |
| [D3CTF2021-8-bit-pub-wp](../raw/web/D3CTF2021-8-bit-pub-wp.md) | node-mysql 对象展开绕过登录，再用 shvl 原型链污染 nodemailer/sendmail 参数 RCE。 | [node-and-prototype.md](node-and-prototype.md) |
| [D3CTF2021-happy-valentines-day-wp](../raw/web/D3CTF2021-happy-valentines-day-wp.md) | 模板/SSTI 语义是主入口，先确认 Java/模板上下文和可达命令执行对象。 | [php-lfi-ssti-ssrf-and-type-juggling.md](php-lfi-ssti-ssrf-and-type-juggling.md) |
| [D3CTF2021-non-rce-wp](../raw/web/D3CTF2021-non-rce-wp.md) | JDBC URL 黑名单竞态后触发 MySQL Connector/J 反序列化和 AspectJWeaver 文件写 gadget。 | [php-java-python-deserialization.md](php-java-python-deserialization.md) |
| [D3CTF2021-pool-calc-wp](../raw/web/D3CTF2021-pool-calc-wp.md) | 多服务计算器分别暴露命令拼接、pickle、Swoole 和 Java RMI 反序列化，先按服务边界拆链。 | [php-java-python-deserialization.md](php-java-python-deserialization.md) |
| [D3CTF2021-real-cloud-serverless-wp](../raw/web/D3CTF2021-real-cloud-serverless-wp.md) | 云函数/Serverless/Kubernetes 内部 API 链，先固定 SSRF 到控制平面的可达边界。 | [artifact-trust-ssrf-to-node-require-rce.md](artifact-trust-ssrf-to-node-require-rce.md) |
| [D3CTF2021-real-cloud-storage-wp](../raw/web/D3CTF2021-real-cloud-storage-wp.md) | 云存储/内部 metadata 或可信服务接口可被 SSRF 触达，先确认凭据和对象读取路径。 | [artifact-trust-ssrf-to-node-require-rce.md](artifact-trust-ssrf-to-node-require-rce.md) |
| [D3CTF2021-shellgen-wp](../raw/web/D3CTF2021-shellgen-wp.md) | token 路径穿越写入模板目录形成 Jinja SSTI，再连 rootless Docker socket 控宿主用户权限。 | [artifact-trust-ssrf-to-node-require-rce.md](artifact-trust-ssrf-to-node-require-rce.md) |
| [D3CTF2022-d3fgo-wp](../raw/web/D3CTF2022-d3fgo-wp.md) | Go/Mongo 管理员登录字段为 interface{}，JSON 可注入 NoSQL 操作符绕过认证。 | [auth-edge-cases-and-protocol-bypasses.md](auth-edge-cases-and-protocol-bypasses.md) |
| [D3CTF2022-ezsql-wp](../raw/web/D3CTF2022-ezsql-wp.md) | SQL 注入或过滤 oracle 是主线，先比较错误、回显、时间和二阶触发条件。 | [sqli-filter-and-oracle-family.md](sqli-filter-and-oracle-family.md) |
| [D3CTF2022-newest-wordpress-wp](../raw/web/D3CTF2022-newest-wordpress-wp.md) | WordPress/已知组件版本差异是主入口，先定位插件/核心 CVE 与补丁边界。 | [known-cves-and-n-day-exploits.md](known-cves-and-n-day-exploits.md) |
| [D3CTF2022-shorter-wp](../raw/web/D3CTF2022-shorter-wp.md) | Java 原生反序列化入口明确，关键是压缩 TemplatesImpl/ROME payload 长度。 | [php-java-python-deserialization.md](php-java-python-deserialization.md) |
| [D3CTF2023-d3cloud-wp](../raw/web/D3CTF2023-d3cloud-wp.md) | laravel-admin 上传 zip 后拼接 unzip 命令，先控制文件名进入命令执行。 | [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md) |
| [D3CTF2023-d3dolphin-wp](../raw/web/D3CTF2023-d3dolphin-wp.md) | remember-me cookie 可伪造管理员登录，后续利用日志包含执行 PHP。 | [auth-bypass-cookies-and-hidden-routes.md](auth-bypass-cookies-and-hidden-routes.md) |
| [D3CTF2023-d3forest-wp](../raw/web/D3CTF2023-d3forest-wp.md) | Forest SSRF 响应会被 Fastjson 自动反序列化，先构造文件读取 gadget 和盲注反馈。 | [php-java-python-deserialization.md](php-java-python-deserialization.md) |
| [D3CTF2023-d3go-wp](../raw/web/D3CTF2023-d3go-wp.md) | Go embed/static 目录穿越拿源码，再用 Gorm 软删除注入和 zip 解压覆盖 self-update。 | [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md) |
| [D3CTF2023-d3icu-wp](../raw/web/D3CTF2023-d3icu-wp.md) | Redis session 反序列化到 CommonsCollections，再利用旧 Chromium CVE 完成浏览器侧 RCE。 | [known-cves-and-n-day-exploits.md](known-cves-and-n-day-exploits.md) |
| [D3CTF2023-d3node-wp](../raw/web/D3CTF2023-d3node-wp.md) | Node/Mongo NoSQL 注入、URL-like 任意读和 npm prepack 脚本串成 RCE。 | [node-and-prototype.md](node-and-prototype.md) |
| [D3CTF2023-egg4shell-wp](../raw/web/D3CTF2023-egg4shell-wp.md) | Egg.js SSRF 到 cluster 通信后触发 watcher 原型污染和 Mongo Code 反序列化竞态。 | [node-and-prototype.md](node-and-prototype.md) |
| [D3CTF2023-escape-plan-wp](../raw/web/D3CTF2023-escape-plan-wp.md) | Python Web 沙箱过滤数字/字母，先用 Unicode 同形字符、len 和切片构造可执行表达式。 | [pyjails.md](pyjails.md) |
| [D3CTF2023-ezjava-wp](../raw/web/D3CTF2023-ezjava-wp.md) | Sofa Hessian 和 Fastjson/Java 原生反序列化链组合，先绕黑名单再控制动态配置。 | [php-java-python-deserialization.md](php-java-python-deserialization.md) |
| [D3CTF2025-d3invitation-wp](../raw/web/D3CTF2025-d3invitation-wp.md) | MinIO STS session token 是 JWT，object_name 可注入 policy 影响对象访问权限。 | [auth-jwt.md](auth-jwt.md) |
| [D3CTF2025-d3jtar-wp](../raw/web/D3CTF2025-d3jtar-wp.md) | jtar 文件名 Unicode 到 ASCII 转换造成后缀变化，先借备份/恢复把上传文件变成 JSP。 | [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md) |
| [D3CTF2025-d3model-wp](../raw/web/D3CTF2025-d3model-wp.md) | Keras .keras 模型 load_model 反序列化 CVE，先控制 config.json 的类/函数解析路径。 | [php-java-python-deserialization.md](php-java-python-deserialization.md) |
| [D3CTF2025-tidy-quic-wp](../raw/web/D3CTF2025-tidy-quic-wp.md) | HTTP/3/QUIC ContentLength 与实际 body 不一致叠加脏缓冲池，先复现请求体污染绕 WAF。 | [parser-wrapper-and-legacy-ssrf-tricks.md](parser-wrapper-and-legacy-ssrf-tricks.md) |

## 原始资料
- [web-first-pass-triage-and-chain-patterns.md](../raw/web/web-first-pass-triage-and-chain-patterns.md)
