---
type: family
tags: [web, family, triage]
skills: [ctf-web]
raw:
  - ../raw/web/web-first-pass-triage-and-chain-patterns.md
updated: 2026-07-06
---

# First-Pass Triage and Chain Patterns

## 作用边界

本页是 Web 方向首轮 family。它只负责把题面、源码、请求行为和初始回显分流到更具体的 family / technique；不承载具体 payload 字典，也不替代 raw WP。

首轮目标是回答四个问题：

1. 输入从哪里进入：URL、Header、Cookie、body、文件、模板、浏览器、内部 API 还是第三方协议。
2. 哪一层先解释它：代理、框架路由、认证中间件、解析器、模板引擎、数据库、浏览器或内部 worker。
3. 结果能观察到什么：回显、错误、状态码、时间、跳转、文件、token、bot 行为或二段请求。
4. 最短链路是什么：读源码/secret、绕认证、访问内部服务、执行代码、浏览器外带或业务状态竞争。

## 识别信号

- 题面或源码暴露 HTTP 路由、认证状态、用户可控请求字段、上传/渲染/转换功能、bot 访问或内部 worker。
- 同一输入可能被代理、框架、认证中间件、数据库、模板、浏览器或后台任务以不同语义解释。
- 初始反馈来自 Web 层可观测差异：状态码、错误文本、响应长度、跳转、时间、cookie、文件变化、外连或 bot 回访。
- 需要先判断主障碍是授权边界、服务端解释链、浏览器上下文、内部服务信任链、已知组件漏洞还是业务状态竞争。

## 最小证据

- 保存一条正常业务请求和一条最小异常请求，标出 method、path、query、header、cookie、content-type 和 body 差异。
- 明确输入第一次被解释的位置：代理、路由、中间件、模板、数据库、解析器、浏览器、worker、runner 或内部 API。
- 记录可观测反馈：状态码、错误文本、响应长度、时间、跳转、文件变化、外连、bot 访问或队列状态。
- 在选下一跳前，先说明目标是读源码/secret、绕认证、访问内网、执行代码、浏览器外带还是业务状态竞争。

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
| 远控/relay/client backend、第三方协议字段进入内网服务 | [protocol-relay-and-internal-service-injection.md](protocol-relay-and-internal-service-injection.md) | 控制面 key 校验是否覆盖所有消息路径，目标是否会主动连接本机 Redis/Docker/API。 |
| XSS、DOM、admin bot、CSP、XS-Leak、cache/MIME | [xss-dom-and-browser-tricks.md](xss-dom-and-browser-tricks.md)、[csp-xsleak-and-browser-exfiltration.md](csp-xsleak-and-browser-exfiltration.md) | 浏览器上下文、CSP/同源限制、外带 oracle 和 bot 触发条件。 |
| 已知组件、CVE、WordPress/Chromium/ExifTool/框架版本 | [known-cves-and-n-day-exploits.md](known-cves-and-n-day-exploits.md) | 版本、补丁边界、利用前置条件和是否需要串其它漏洞。 |
| Jail、表达式沙箱、Python/Web 混合过滤 | [pyjails.md](pyjails.md) | 过滤发生在语言、表达式、对象图还是 Web 包装层。 |

## 失败后 pivot

- 一个请求没有差异：固定正常业务流，逐个改变 method、path、header、cookie、body、content-type 和编码层。
- 有错误但不能外带：优先找源码、配置、secret、日志、debug endpoint 或内部回显，不要急着堆 RCE payload。
- 有 SSRF/内部访问但无响应：转 blind oracle，观察时间、DNS、缓存、文件写入、队列状态或二段请求。
- 有 XSS 但拿不到数据：转 CSP/XS-Leak，构造可重复的浏览器 oracle。
- 已经 RCE 但权限低：回到文件、环境变量、容器/云 metadata、内部服务和凭据复用。

## 常见误判

- 只凭题名或技术栈选路线，没有先确认输入在哪一层被解释。
- 把前端限制当成服务端授权，或把 UI 报错当作真实权限边界。
- 看到 SSRF/XSS/RCE 字样就直接套 payload，没有先固定可观测 oracle。
- 多阶段链路只记录最后一步，丢失源码泄露、secret 恢复、路径定位或 bot 触发条件。

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
| [WMCTF2025-guess-wp](../raw/web/WMCTF2025-guess-wp.md) | Flask 接口连续泄露 `random.Random().getrandbits(32)` 输出，再用预测的 key 进入 Python `eval`；先恢复 MT 状态，再确认表达式沙箱逃逸。 | [mt-lcg-and-seed-recovery.md](mt-lcg-and-seed-recovery.md)、[pyjails.md](pyjails.md) |
| [WMCTF2025-pdf2text-wp](../raw/web/WMCTF2025-pdf2text-wp.md) | pdfminer Type0 字体 `/Encoding` 可路径穿越到上传目录，CMap gzip pickle 被反序列化；先证明 parser 内部对象能越过文件名过滤。 | [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md)、[php-java-python-deserialization.md](php-java-python-deserialization.md)、[parser-wrapper-and-legacy-ssrf-tricks.md](parser-wrapper-and-legacy-ssrf-tricks.md) |
| [WMCTF2025-rustdesk-change-client-backend-wp](../raw/web/WMCTF2025-rustdesk-change-client-backend-wp.md) | 远控协议 relay 消息绕过 key 校验，让受控端把可控字段发给本机 Redis，再串主从同步模块 RCE。 | [protocol-relay-and-internal-service-injection.md](protocol-relay-and-internal-service-injection.md) |
| [ACTF2026-12307-wp](../raw/web/ACTF2026-12307-wp.md) | SSO continuation 只做字符串包含建立 settlement 信任，随后 ORDER BY 盲注泄露 `claim_salt`，最终用 duplicate-key print plan 混淆进入私有打印通道。 | [auth-bypass-cookies-and-hidden-routes.md](auth-bypass-cookies-and-hidden-routes.md)、[sqli-filter-and-oracle-family.md](sqli-filter-and-oracle-family.md)、[json-duplicate-key-hmac-parser-differential.md](json-duplicate-key-hmac-parser-differential.md) |
| [ACTF2026-aaa26-wp](../raw/web/ACTF2026-aaa26-wp.md) | Mongo `$regex` 盲注恢复 reviewer invite code，vm2 里用 Buffer slab 泄露 JWT secret，伪造 admin 后上传伪 PDF/SVG 让 ImageMagick `text:/flag` 渲染。 | [sqli-filter-and-oracle-family.md](sqli-filter-and-oracle-family.md)、[auth-jwt.md](auth-jwt.md)、[parser-wrapper-and-legacy-ssrf-tricks.md](parser-wrapper-and-legacy-ssrf-tricks.md) |
| [ACTF2026-gomysql-wp](../raw/web/ACTF2026-gomysql-wp.md) | `/calc` 多语句 MariaDB 注入可写 UDF 并下载 root Go 二进制；最终利用 `/draw` 自写模板的变量替换和 safe/unsafe 顺序执行 `run('cat /flag')`。 | [sqli-upload-deser-and-command-rce.md](sqli-upload-deser-and-command-rce.md)、[ruby-php-upload-and-ssti-rce.md](ruby-php-upload-and-ssti-rce.md) |
| [ACTF2026-real-dlsite-wp](../raw/web/ACTF2026-real-dlsite-wp.md) | go-drive `CleanPath`/`Join` 不一致先拿 app RCE，再串 PHP open_basedir check/use race 和 StorageBox setuid TOCTOU 读 root flag。 | [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md)、[race-condition-and-concurrency-exploits.md](race-condition-and-concurrency-exploits.md) |
| [ACTF2026-tinj-wp](../raw/web/ACTF2026-tinj-wp.md) | JPHP/Jetty 字符字节转换绕过路由后进入 eval、JAR/classloader 和内存马链，先确认运行时边界。 | [php-lfi-ssti-ssrf-and-type-juggling.md](php-lfi-ssti-ssrf-and-type-juggling.md) |
| [TAMUctf2021-API-2-The-SeQueL-wp](../raw/web/TAMUctf2021-API-2-The-SeQueL-wp.md) | PostgreSQL 字符串型 UNION 注入，错误信息暴露列数和第三列 `contlevel` 枚举；用合法枚举值占位后读 `users` 表。 | [sqli-filter-and-oracle-family.md](sqli-filter-and-oracle-family.md) |
| [TAMUctf2022-Serial-Killer-wp](../raw/web/TAMUctf2022-Serial-Killer-wp.md) | Cookie 解码后是 PHP 序列化对象，`file` 属性进入 include；关键是同步长度字段并利用 URL 解码顺序绕过 `../` 过滤。 | [php-java-python-deserialization.md](php-java-python-deserialization.md)、[php-lfi-ssti-ssrf-and-type-juggling.md](php-lfi-ssti-ssrf-and-type-juggling.md) |
| [HGAME2026-博丽神社的绘马挂-wp](../raw/web/HGAME2026-博丽神社的绘马挂-wp.md) | 留言触发 bot 存储型 XSS，归档搜索支持 JSONP callback；让管理员同源上下文查询私有归档并回传。 | [xss-dom-and-browser-tricks.md](xss-dom-and-browser-tricks.md) |
| [HGAME2026-魔理沙的魔法目录-wp](../raw/web/HGAME2026-魔理沙的魔法目录-wp.md) | 阅读时长由前端周期性上报 `record?time=`，后端信任累计值；直接重放大时间值绕过等待。 | [auth-edge-cases-and-protocol-bypasses.md](auth-edge-cases-and-protocol-bypasses.md) |
| [HGAME2026-文文新闻-wp](../raw/web/HGAME2026-文文新闻-wp.md) | Vite 任意文件读取先拿路径和后端源码，再利用代理不清理 TE、后端只信 Content-Length 的 CL/TE 差异做请求走私。 | [known-cves-and-n-day-exploits.md](known-cves-and-n-day-exploits.md)、[parser-wrapper-and-legacy-ssrf-tricks.md](parser-wrapper-and-legacy-ssrf-tricks.md) |
| [HGAME2026-babyweb-wp](../raw/web/HGAME2026-babyweb-wp.md) | PHP 上传允许 .php/.htaccess 并可落 WebShell，先证明可控文件内容、解析规则和内网 pivot。 | [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md) |
| [HGAME2026-easyuu-wp](../raw/web/HGAME2026-easyuu-wp.md) | 可控目录列表找到更新包源码，上传后门 `path1` 可覆盖 `./update/easyuu`，再借自更新机制执行自定义程序。 | [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md)、[artifact-trust-ssrf-to-node-require-rce.md](artifact-trust-ssrf-to-node-require-rce.md) |
| [HGAME2026-ezcc-wp](../raw/web/HGAME2026-ezcc-wp.md) | `userInfo` cookie Base64 后直接 Java 反序列化，commons-collections 3.2.1 黑名单只禁 InvokerTransformer；换 CC6/CC3 TemplatesImpl 链。 | [php-java-python-deserialization.md](php-java-python-deserialization.md) |
| [HGAME2026-my-little-assistant-wp](../raw/web/HGAME2026-my-little-assistant-wp.md) | MCP 工具只暴露 `py_request`，但 Playwright 禁用 Web 安全后外部页面可请求 localhost MCP JSON-RPC 调用 `py_eval`。 | [workflow-runner-internal-api-chain.md](workflow-runner-internal-api-chain.md)、[protocol-relay-and-internal-service-injection.md](protocol-relay-and-internal-service-injection.md) |
| [HGAME2026-my-monitor-wp](../raw/web/HGAME2026-my-monitor-wp.md) | Go `sync.Pool` 中 JSON bind 错误路径不 reset，低权限请求污染 `Args`，管理员 `bash -c` 执行时继承残留字段。 | [race-condition-and-concurrency-exploits.md](race-condition-and-concurrency-exploits.md)、[xml-command-and-graphql-injection.md](xml-command-and-graphql-injection.md) |
| [HGAME2026-vidarshop-wp](../raw/web/HGAME2026-vidarshop-wp.md) | JWT 头是干扰项，真实身份由可预测 uid 决定；拿到管理员语义后用 `__proto__`/constructor 污染余额状态。 | [auth-edge-cases-and-protocol-bypasses.md](auth-edge-cases-and-protocol-bypasses.md)、[node-and-prototype.md](node-and-prototype.md) |
| [LilacCTF2026-checkin-wp](../raw/web/LilacCTF2026-checkin-wp.md) | Python eval jail 经过 IDNA/正则过滤后仍保留 `vars()`、`dir()` 和可变 `LockedList`；原地 pop/append 修改状态。 | [pyjails.md](pyjails.md) |
| [LilacCTF2026-nailong-wp](../raw/web/LilacCTF2026-nailong-wp.md) | Pickle/PyTorch 模型加载执行链和扫描绕过，先确认反序列化入口、ZIP/CRC 信任边界和命令执行点。 | [php-java-python-deserialization.md](php-java-python-deserialization.md) |
| [LilacCTF2026-playground-wp](../raw/web/LilacCTF2026-playground-wp.md) | 浏览器执行模型或 bot 行为可控，先确认 sink、CSP、触发上下文和 exfil 通道。 | [xss-dom-and-browser-tricks.md](xss-dom-and-browser-tricks.md) |
| [LilacCTF2026-safe-sql-wp](../raw/web/LilacCTF2026-safe-sql-wp.md) | PostgreSQL `E'...'` 只过滤引号时可用反斜杠让 password 逃逸成多语句；再借 BRIN autosummarize CVE 提权到 `COPY FROM PROGRAM`。 | [sqli-filter-and-oracle-family.md](sqli-filter-and-oracle-family.md)、[known-cves-and-n-day-exploits.md](known-cves-and-n-day-exploits.md) |
| [NCTF2026-n-minsite-wp](../raw/web/NCTF2026-n-minsite-wp.md) | MaxSite 源码可由 `/?key` 获取，后台上传入口只校验登录态；上传 HTML/JS 后诱导 bot 读取后台 flag。 | [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md)、[xss-dom-and-browser-tricks.md](xss-dom-and-browser-tricks.md) |
| [NCTF2026-n-rustpica-wp](../raw/web/NCTF2026-n-rustpica-wp.md) | 静态调试目录泄露后台凭据后，Rust/Serde `untagged enum` 和未知字段忽略导致工作流状态绕过。 | [auth-edge-cases-and-protocol-bypasses.md](auth-edge-cases-and-protocol-bypasses.md)、[json-duplicate-key-hmac-parser-differential.md](json-duplicate-key-hmac-parser-differential.md) |
| [NCTF2026-openshell-wp](../raw/web/NCTF2026-openshell-wp.md) | OpenCode `/find` 把 `pattern` 拼入 ripgrep 参数并用 Bun shell `raw` 执行；用 bot 访问 GET 路由触发命令注入。 | [ruby-php-upload-and-ssti-rce.md](ruby-php-upload-and-ssti-rce.md)、[known-cves-and-n-day-exploits.md](known-cves-and-n-day-exploits.md) |
| [RCTF2025-514s-heart-wp](../raw/web/RCTF2025-514s-heart-wp.md) | Koishi 插件静态路由目录穿越先读配置拿后台密码，再用配置 `${{ ... }}` 表达式执行命令。 | [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md)、[ruby-php-upload-and-ssti-rce.md](ruby-php-upload-and-ssti-rce.md) |
| [RCTF2025-auth-wp](../raw/web/RCTF2025-auth-wp.md) | Node IdP 中 `parseInt(false)` 与 MySQL `TINYINT` 存储语义不一致可注册 admin，再利用 SP XML Signature Wrapping 让业务读取未签名 Assertion。 | [auth-edge-cases-and-protocol-bypasses.md](auth-edge-cases-and-protocol-bypasses.md)、[oauth-saml-cors-and-cicd.md](oauth-saml-cors-and-cicd.md) |
| [RCTF2025-514-wp](../raw/web/RCTF2025-514-wp.md) | Bot 插件把用户文本写入 Puppeteer 截图页面，且渲染上下文是 `file://`；注入 iframe 读取本地 flag 并用覆盖层让截图回显。 | [xss-dom-and-browser-tricks.md](xss-dom-and-browser-tricks.md) |
| [RCTF2025-author-plus-wp](../raw/web/RCTF2025-author-plus-wp.md) | author meta 属性注入过滤了常规空白但漏 `%0b/%0c`，用 CSP meta 压制防护脚本，并借 popover `onbeforetoggle` 在 bot 点击时触发外带。 | [xss-dom-and-browser-tricks.md](xss-dom-and-browser-tricks.md)、[csp-xsleak-and-browser-exfiltration.md](csp-xsleak-and-browser-exfiltration.md) |
| [RCTF2025-author-wp](../raw/web/RCTF2025-author-wp.md) | 用户名进入 `<meta name=author>` 属性，可注入 CSP 限制 `xss-shield.js`，再在文章正文用 `img onerror` 外带 cookie。 | [xss-dom-and-browser-tricks.md](xss-dom-and-browser-tricks.md)、[csp-xsleak-and-browser-exfiltration.md](csp-xsleak-and-browser-exfiltration.md) |
| [RCTF2025-maybe-easy-wp](../raw/web/RCTF2025-maybe-easy-wp.md) | Hessian 反序列化受包名前缀白名单限制；用 Spring/JNDI 辅助类和 PriorityQueue `compareTo` 触发 gadget。 | [php-java-python-deserialization.md](php-java-python-deserialization.md) |
| [RCTF2025-photographer-wp](../raw/web/RCTF2025-photographer-wp.md) | SQLite `SELECT *` join 中同名 `type` 字段覆盖用户权限字段，上传图片 MIME type 可写成 `-1` 绕过管理员判断。 | [auth-edge-cases-and-protocol-bypasses.md](auth-edge-cases-and-protocol-bypasses.md)、[sqli-filter-and-oracle-family.md](sqli-filter-and-oracle-family.md) |
| [RCTF2025-rootkb-dash-wp](../raw/web/RCTF2025-rootkb-dash-wp.md) | 工具沙箱可连默认口令 Redis，Celery token key 使用 pickle 且类加载限制不足，写恶意 pickle 等管理员会话触发。 | [php-java-python-deserialization.md](php-java-python-deserialization.md)、[protocol-relay-and-internal-service-injection.md](protocol-relay-and-internal-service-injection.md) |
| [RCTF2025-rootkb-wp](../raw/web/RCTF2025-rootkb-wp.md) | 工具沙箱目录可写且运行时固定 `LD_PRELOAD=sandbox.so`，覆盖共享库后靠 constructor 把 flag 移到 Web 静态目录。 | [source-backdoors-and-restricted-shell-tricks.md](source-backdoors-and-restricted-shell-tricks.md)、[ruby-php-upload-and-ssti-rce.md](ruby-php-upload-and-ssti-rce.md) |
| [RCTF2025-ultimatefreeloader-wp](../raw/web/RCTF2025-ultimatefreeloader-wp.md) | 下单和退款使用不同 Redis lock key，先并发触发购买/退款窗口并验证余额、优惠券和商品状态不互斥。 | [race-condition-and-concurrency-exploits.md](race-condition-and-concurrency-exploits.md) |
| [Spirit2026-5-artifact-relay-wp](../raw/web/Spirit2026-5-artifact-relay-wp.md) | 双重 URL 解码读源码/key/token，解密 runner token 后把 artifact 标记 trusted，触发 Node `require()` 规则文件 RCE。 | [artifact-trust-ssrf-to-node-require-rce.md](artifact-trust-ssrf-to-node-require-rce.md) |
| [Spirit2026-5-flowforge-wp](../raw/web/Spirit2026-5-flowforge-wp.md) | duplicate-key HMAC parser 差异提升权限后导入自定义 workflow node，runner 携带内部 token 请求 GraphQL/MCP debug export。 | [json-duplicate-key-hmac-parser-differential.md](json-duplicate-key-hmac-parser-differential.md)、[workflow-runner-internal-api-chain.md](workflow-runner-internal-api-chain.md) |
| [Spirit2026-5-trust-collapse-wp](../raw/web/Spirit2026-5-trust-collapse-wp.md) | 路径视角差异绕过管理/内部 ACL，修改构建配置并泄露签名材料，最终伪造 signed internal request。 | [path-confusion-to-signed-internal-request-chain.md](path-confusion-to-signed-internal-request-chain.md) |
| [SUCTF2026-cmsAgainWP](../raw/web/SUCTF2026-cmsAgainWP.md) | 前台购物车 Cookie 反序列化污染 ProductID 形成 SQLi，盲注拿后台后写 `{~...}` 模板片段触发 ThinkPHP PHP 执行。 | [sqli-upload-deser-and-command-rce.md](sqli-upload-deser-and-command-rce.md)、[ruby-php-upload-and-ssti-rce.md](ruby-php-upload-and-ssti-rce.md) |
| [SUCTF2026-jdbc-masterWP](../raw/web/SUCTF2026-jdbc-masterWP.md) | Unicode 长 `ſ` 绕过 `/suctf` 路径过滤后，可控 JDBC driver/URL 走 Kingbase `ConfigurePath` 与 Spring XML beans 无外连加载。 | [auth-edge-cases-and-protocol-bypasses.md](auth-edge-cases-and-protocol-bypasses.md)、[xml-command-and-graphql-injection.md](xml-command-and-graphql-injection.md) |
| [SUCTF2026-Note_revWP](../raw/web/SUCTF2026-Note_revWP.md) | `search.php?q=` 进入内联 JS 字符串且可用 `</script>` 闭合；让 bot 访问内网同源搜索页后 fetch 管理员 notes 并外带。 | [xss-dom-and-browser-tricks.md](xss-dom-and-browser-tricks.md) |
| [SUCTF2026-NoteWP](../raw/web/SUCTF2026-NoteWP.md) | `/bot/` 可请求 `127.0.0.1:80` 并把内部响应 `Set-Cookie` 透传给攻击者，直接泄露 bot/admin 的 PHPSESSID 后读取 notes。 | [auth-bypass-cookies-and-hidden-routes.md](auth-bypass-cookies-and-hidden-routes.md)、[polyglot-url-tricks-and-ssrf-leaks.md](polyglot-url-tricks-and-ssrf-leaks.md) |
| [SUCTF2026-sqliWP](../raw/web/SUCTF2026-sqliWP.md) | `/api/query` 注入前必须复现 Go WASM 前端签名和浏览器指纹字段；签名过后用 PostgreSQL 字符串拼接构造布尔盲注。 | [sqli-filter-and-oracle-family.md](sqli-filter-and-oracle-family.md)、[game-state-websocket-and-wasm.md](game-state-websocket-and-wasm.md) |
| [SUCTF2026-uriWP](../raw/web/SUCTF2026-uriWP.md) | 后端 SSRF 只检查解析时 IP，不绑定后续连接结果；DNS rebinding 命中 Docker Remote API 后挂载宿主根目录执行 `/readflag`。 | [polyglot-url-tricks-and-ssrf-leaks.md](polyglot-url-tricks-and-ssrf-leaks.md)、[protocol-relay-and-internal-service-injection.md](protocol-relay-and-internal-service-injection.md) |
| [SUCTF2026-wmsWP](../raw/web/SUCTF2026-wmsWP.md) | JEECG `/rest/*` 白名单放行且 Spring `params` 可由 POST body 命中后台方法；模板 ZIP 解压目录穿越写 JSP。 | [auth-bypass-cookies-and-hidden-routes.md](auth-bypass-cookies-and-hidden-routes.md)、[path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md) |
| [VNCTF2026-black-coffee-wp](../raw/web/VNCTF2026-black-coffee-wp.md) | Java 原生反序列化 gadget 链，先确认编码层、触发类、TemplatesImpl/POJONode 等可用 gadget。 | [php-java-python-deserialization.md](php-java-python-deserialization.md) |
| [VNCTF2026-i-really-really-really-and-revenge-wp](../raw/web/VNCTF2026-i-really-really-really-and-revenge-wp.md) | Pyjail 封数字、字符串和 builtins 后，仍可用布尔造数、类元信息造字符串、`object.__subclasses__()` 找全局模块，并用异常外带。 | [pyjails.md](pyjails.md) |
| [VNCTF2026-i-really-really-really-ultimate-wp](../raw/web/VNCTF2026-i-really-really-really-ultimate-wp.md) | Unicode 绕过被补后，生成器 `gi_frame.f_back` 仍能回溯栈帧恢复 `builtins`，再拼 `/flag` 并用断言报错泄露。 | [pyjails.md](pyjails.md) |
| [VNCTF2026-markdown2world-wp](../raw/web/VNCTF2026-markdown2world-wp.md) | Pandoc 转 docx 时会读取 Markdown 图片目标并嵌入 `word/media/`；构造本地资源引用后解压 docx 取回文件内容。 | [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md)、[parser-wrapper-and-legacy-ssrf-tricks.md](parser-wrapper-and-legacy-ssrf-tricks.md) |
| [VNCTF2026-more-black-coffee-wp](../raw/web/VNCTF2026-more-black-coffee-wp.md) | Java 反序列化继续加载内存马，先确认 FilterShell 注入点、触发链和容器运行时边界。 | [php-java-python-deserialization.md](php-java-python-deserialization.md) |
| [VNCTF2026-signin-wp](../raw/web/VNCTF2026-signin-wp.md) | PHP `include` 可控且过滤 `php://filter` 常见写法，先用短 `data:,<?=...` 伪协议构造执行内容。 | [php-lfi-ssti-ssrf-and-type-juggling.md](php-lfi-ssti-ssrf-and-type-juggling.md) |
| [VNCTF2026-web-pentest-wp](../raw/web/VNCTF2026-web-pentest-wp.md) | 前端混合加密登录把 SM4 `key/iv` 经 SM2 封装给服务端；若封装值可固定复用，只需重算业务密文和 MD5 签名即可爆破。 | [auth-edge-cases-and-protocol-bypasses.md](auth-edge-cases-and-protocol-bypasses.md)、[web-tooling.md](web-tooling.md) |
| [D3CTF2019-babyxss-wp](../raw/web/D3CTF2019-babyxss-wp.md) | XSS/CSP/browser bot 是主线，先确认 sink、CSP 约束和可用 exfil 通道。 | [xss-dom-and-browser-tricks.md](xss-dom-and-browser-tricks.md) |
| [D3CTF2019-d3guestbook-wp](../raw/web/D3CTF2019-d3guestbook-wp.md) | HTML 白名单、JSONP callback 和 CSRF token/sessionid 绑定组合，先构造表单劫持拿管理员 token。 | [xss-dom-and-browser-tricks.md](xss-dom-and-browser-tricks.md) |
| [D3CTF2019-easyweb-wp](../raw/web/D3CTF2019-easyweb-wp.md) | 二次 SQL 注入控制 Smarty 模板，再经 php:phar 触发反序列化 POP 链。 | [sqli-upload-deser-and-command-rce.md](sqli-upload-deser-and-command-rce.md) |
| [D3CTF2019-ezts-wp](../raw/web/D3CTF2019-ezts-wp.md) | Sequelize JSON path 注入进入后台后，再用 lodash.defaultsDeep 原型污染 EJS RCE。 | [node-and-prototype.md](node-and-prototype.md) |
| [D3CTF2019-ezupload-wp](../raw/web/D3CTF2019-ezupload-wp.md) | 反序列化任意文件写结合 glob 爆破真实上传路径和 .htaccess 解析绕过。 | [php-java-python-deserialization.md](php-java-python-deserialization.md)、[path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md) |
| [D3CTF2019-fake-onelinephp-wp](../raw/web/D3CTF2019-fake-onelinephp-wp.md) | Windows PHP 文件包含可走 WebDAV/SMB 远程包含，再从 git 和内网凭据继续 pivot。 | [php-lfi-ssti-ssrf-and-type-juggling.md](php-lfi-ssti-ssrf-and-type-juggling.md) |
| [D3CTF2019-showhub-wp](../raw/web/D3CTF2019-showhub-wp.md) | insert 注入可用 on duplicate key update 改管理员密码，后续还需 request smuggling 绕过内网 IP 限制。 | [sqli-upload-deser-and-command-rce.md](sqli-upload-deser-and-command-rce.md) |
| [D3CTF2021-8-bit-pub-wp](../raw/web/D3CTF2021-8-bit-pub-wp.md) | node-mysql 对象展开绕过登录，再用 shvl 原型链污染 nodemailer/sendmail 参数 RCE。 | [node-and-prototype.md](node-and-prototype.md) |
| [D3CTF2021-happy-valentines-day-wp](../raw/web/D3CTF2021-happy-valentines-day-wp.md) | 模板/SSTI 语义是主入口，先确认 Java/模板上下文和可达命令执行对象。 | [php-lfi-ssti-ssrf-and-type-juggling.md](php-lfi-ssti-ssrf-and-type-juggling.md) |
| [D3CTF2021-non-rce-wp](../raw/web/D3CTF2021-non-rce-wp.md) | JDBC URL 黑名单竞态后触发 MySQL Connector/J 反序列化和 AspectJWeaver 文件写 gadget。 | [php-java-python-deserialization.md](php-java-python-deserialization.md) |
| [D3CTF2021-pool-calc-wp](../raw/web/D3CTF2021-pool-calc-wp.md) | 多服务计算器分别暴露命令拼接、pickle、Swoole 和 Java RMI 反序列化，先按服务边界拆链。 | [php-java-python-deserialization.md](php-java-python-deserialization.md) |
| [D3CTF2021-real-cloud-serverless-wp](../raw/cloud-infra/D3CTF2021-real-cloud-serverless-wp.md) | 云函数/Serverless/Kubernetes 内部 API 链，先固定 SSRF 到控制平面的可达边界。 | [artifact-trust-ssrf-to-node-require-rce.md](artifact-trust-ssrf-to-node-require-rce.md) |
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
| [D3CTF2025-d3invitation-wp](../raw/cloud-infra/D3CTF2025-d3invitation-wp.md) | MinIO STS session token 是 JWT，object_name 可注入 policy 影响对象访问权限。 | [auth-jwt.md](auth-jwt.md) |
| [D3CTF2025-d3jtar-wp](../raw/web/D3CTF2025-d3jtar-wp.md) | jtar 文件名 Unicode 到 ASCII 转换造成后缀变化，先借备份/恢复把上传文件变成 JSP。 | [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md) |
| [D3CTF2025-d3model-wp](../raw/web/D3CTF2025-d3model-wp.md) | Keras .keras 模型 load_model 反序列化 CVE，先控制 config.json 的类/函数解析路径。 | [known-cves-and-n-day-exploits.md](known-cves-and-n-day-exploits.md)、[php-java-python-deserialization.md](php-java-python-deserialization.md) |
| [D3CTF2025-tidy-quic-wp](../raw/web/D3CTF2025-tidy-quic-wp.md) | HTTP/3/QUIC ContentLength 与实际 body 不一致叠加脏缓冲池，先复现请求体污染绕 WAF。 | [parser-wrapper-and-legacy-ssrf-tricks.md](parser-wrapper-and-legacy-ssrf-tricks.md) |
| [RCTF2025-signin-wp](../raw/web/RCTF2025-signin-wp.md) | 前端源码把完成度作为 `score` 参数提交，先读业务逻辑并直接验证阈值触发条件。 | 跨页补入 |

## 原始资料
- [web-first-pass-triage-and-chain-patterns.md](../raw/web/web-first-pass-triage-and-chain-patterns.md)
