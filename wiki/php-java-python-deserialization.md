---
type: family
tags: [web, family, deserialization, pickle, java, php, dotnet]
skills: [ctf-web]
raw:
  - ../raw/web/php-java-python-deserialization.md
  - ../raw/web/WMCTF2025-pdf2text-wp.md
  - ../raw/web/D3CTF2019-ezupload-wp.md
  - ../raw/web/D3CTF2025-d3model-wp.md
updated: 2026-07-06
---

# PHP, Java and Python Deserialization

## 作用边界

本页是 Web 反序列化 family，覆盖 Java ysoserial/XMLDecoder/Castor、Python pickle/Werkzeug cookie/library-internal pickle、.NET Json.NET TypeNameHandling、PHP unserialize/SoapClient、session/cookie 包装、STOP opcode 拼接和少量 TOCTOU race。

它不是一个统一 payload 模板。首轮要先判断序列化格式、触发入口、gadget 可用性、回显方式和链路目标，再决定是 DNS 探测、文件读、SSRF、命令执行、cookie forgery 还是 parser 二段加载。

## 识别信号

- 请求、Cookie、session、上传文件、备份导入、XML/JSON 字段或库内部数据文件被服务端反序列化。
- 出现 Java 类名、pickle opcodes、PHP `O:<len>:"Class"`、`.NET $type`、Werkzeug/Flask cookie、XMLDecoder、Castor `xsi:type`。
- 需要 gadget chain、SECRET_KEY、类路径、过滤器长度变化、二次 URL 编码或协议回连。
- 成功信号可能是 DNS/JRMP 回连、异常栈、文件读、HTTP SSRF、命令输出、cookie 权限变化或 parser 触发的内部加载。

## 最小证据

- 确认反序列化 sink 的格式和位置，而不是只看到可疑 Base64。
- 确认类库或运行时：Java 依赖、Python 版本、PHP 类和 magic method、Json.NET 配置、Flask secret。
- 选择安全最小探测：URLDNS/JRMP、无害命令、时间延迟、文件读或 cookie 权限变化。
- 记录包装层：ROT13/Base64/gzip、JSON/XML、session 签名、URL 编码、STOP opcode stripping 或 parser 内部文件名。

## 首轮路由

| 证据形态 | 首轮判断 | 下一跳 |
|---|---|---|
| Java serialized stream、ysoserial gadget、JRMP/DNS 回连 | 先用 URLDNS/JRMP 做无害探测，再按依赖选择 gadget chain | [web-tooling.md](web-tooling.md) |
| XMLDecoder、Castor `xsi:type`、XML 多态类型 | 先确认类型白名单和 setter/constructor 触发，再判断是否能 RCE 或 SSRF | [parser-wrapper-and-legacy-ssrf-tricks.md](parser-wrapper-and-legacy-ssrf-tricks.md), [xml-command-and-graphql-injection.md](xml-command-and-graphql-injection.md) |
| Python pickle、STOP opcode 拼接、ROT13/Base64 包装 | 先反解包装层，再用最小 pickle 副作用验证 sink | [parser-wrapper-and-legacy-ssrf-tricks.md](parser-wrapper-and-legacy-ssrf-tricks.md) |
| Werkzeug/Flask SecureCookie、SECRET_KEY 泄露 | 先验证签名和 session 字段，再伪造身份或触发 pickle 路径 | [auth-bypass-cookies-and-hidden-routes.md](auth-bypass-cookies-and-hidden-routes.md), [auth-jwt.md](auth-jwt.md) |
| .NET Json.NET `TypeNameHandling` | 先确认 `$type` 是否生效，再按可用程序集选 gadget | [pe-and-dotnet.md](pe-and-dotnet.md) |
| PHP unserialize、长度膨胀、SoapClient CRLF、curl LFI | 先校正序列化长度和 magic method，再判断能否 SSRF/CRLF/file read/RCE | [php-lfi-ssti-ssrf-and-type-juggling.md](php-lfi-ssti-ssrf-and-type-juggling.md), [parser-wrapper-and-legacy-ssrf-tricks.md](parser-wrapper-and-legacy-ssrf-tricks.md) |
| 上传文档触发库内部 pickle/gzip/CMap/model 加载 | 先证明 parser 会二段加载，再让内部文件名指向可控数据 | [parser-wrapper-and-legacy-ssrf-tricks.md](parser-wrapper-and-legacy-ssrf-tricks.md) |
| TOCTOU/race 与反序列化链混合 | 先固定竞争窗口，再决定是否转并发利用 | [race-condition-and-concurrency-exploits.md](race-condition-and-concurrency-exploits.md) |

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [WMCTF2025-pdf2text-wp](../raw/web/WMCTF2025-pdf2text-wp.md) | pdfminer 的 `CMapDB._load_data` 会读取 gzip pickle CMap；利用关键是让 PDF `/Encoding` 名称路径穿越到上传目录，并让恶意 gzip pickle 同时通过宽松 PDF 检测。 |
| [D3CTF2019-ezupload-wp](../raw/web/D3CTF2019-ezupload-wp.md) | PHP unserialize 对象在析构阶段形成任意文件写；真正利用前要先解决相对路径失效和上传目录绝对路径定位，再把写文件能力推进到解析执行。 |
| [D3CTF2025-d3model-wp](../raw/web/D3CTF2025-d3model-wp.md) | Web 服务加载不可信 `.keras` 模型，`keras.models.load_model` 读取 `config.json` 后解析类/函数路径；这是模型格式反序列化进入 Python 对象解析的代表案例。 |
| [Bugku-Serial-Killer-wp](../raw/web/Bugku-Serial-Killer-wp.md) | Cookie 解码后是 PHP 序列化对象，`file` 属性进入 include；关键是同步长度字段并利用 URL 解码顺序绕过 `../` 过滤。 |
| [D3CTF2021-non-rce-wp](../raw/web/D3CTF2021-non-rce-wp.md) | JDBC URL 黑名单竞态后触发 MySQL Connector/J 反序列化和 AspectJWeaver 文件写 gadget。 |
| [D3CTF2021-pool-calc-wp](../raw/web/D3CTF2021-pool-calc-wp.md) | 多服务计算器分别暴露命令拼接、pickle、Swoole 和 Java RMI 反序列化，先按服务边界拆链。 |
| [D3CTF2022-shorter-wp](../raw/web/D3CTF2022-shorter-wp.md) | Java 原生反序列化入口明确，关键是压缩 TemplatesImpl/ROME payload 长度。 |
| [D3CTF2023-d3forest-wp](../raw/web/D3CTF2023-d3forest-wp.md) | Forest SSRF 响应会被 Fastjson 自动反序列化，先构造文件读取 gadget 和盲注反馈。 |
| [D3CTF2023-ezjava-wp](../raw/web/D3CTF2023-ezjava-wp.md) | Sofa Hessian 和 Fastjson/Java 原生反序列化链组合，先绕黑名单再控制动态配置。 |
| [HGAME2026-ezcc-wp](../raw/web/HGAME2026-ezcc-wp.md) | `userInfo` cookie Base64 后直接 Java 反序列化，commons-collections 3.2.1 黑名单只禁 InvokerTransformer；换 CC6/CC3 TemplatesImpl 链。 |
| [LilacCTF2026-nailong-wp](../raw/web/LilacCTF2026-nailong-wp.md) | Pickle/PyTorch 模型加载执行链和扫描绕过，先确认反序列化入口、ZIP/CRC 信任边界和命令执行点。 |
| [RCTF2025-maybe-easy-wp](../raw/web/RCTF2025-maybe-easy-wp.md) | Hessian 反序列化受包名前缀白名单限制；用 Spring/JNDI 辅助类和 PriorityQueue `compareTo` 触发 gadget。 |
| [RCTF2025-rootkb-dash-wp](../raw/web/RCTF2025-rootkb-dash-wp.md) | 工具沙箱可连默认口令 Redis，Celery token key 使用 pickle 且类加载限制不足，写恶意 pickle 等管理员会话触发。 |
| [VNCTF2026-black-coffee-wp](../raw/web/VNCTF2026-black-coffee-wp.md) | Java 原生反序列化 gadget 链，先确认编码层、触发类、TemplatesImpl/POJONode 等可用 gadget。 |
| [VNCTF2026-more-black-coffee-wp](../raw/web/VNCTF2026-more-black-coffee-wp.md) | Java 反序列化继续加载内存马，先确认 FilterShell 注入点、触发链和容器运行时边界。 |

## 合并与拆分结论

本页应保留为 family。Java、Python、PHP 和 .NET 的 payload 构造、gadget 选择和探测方式不同，不适合作为单一 technique；但它们共享“服务端恢复对象图时执行副作用”的首轮判断，且多条 Web family 都会把反序列化作为下一跳。

## 常见陷阱

- 看到 Base64 就猜反序列化，没先识别 magic、签名、gzip 或 JSON/XML 包装。
- 直接上 RCE gadget，没用 URLDNS/JRMP/无害副作用确认 sink。
- 忽略类路径和依赖版本，导致 gadget 在本地有效、远程无效。
- PHP 序列化长度被过滤器改变后没有重新计算。
- Cookie 类场景只改字段，不重签名或不验证 SECRET_KEY 来源。

## 关联技巧

- [parser-wrapper-and-legacy-ssrf-tricks.md](parser-wrapper-and-legacy-ssrf-tricks.md)
- [xml-command-and-graphql-injection.md](xml-command-and-graphql-injection.md)
- [php-lfi-ssti-ssrf-and-type-juggling.md](php-lfi-ssti-ssrf-and-type-juggling.md)
- [auth-bypass-cookies-and-hidden-routes.md](auth-bypass-cookies-and-hidden-routes.md)
- [auth-jwt.md](auth-jwt.md)
- [race-condition-and-concurrency-exploits.md](race-condition-and-concurrency-exploits.md)
- [web-tooling.md](web-tooling.md)

## 原始资料

- [php-java-python-deserialization.md](../raw/web/php-java-python-deserialization.md)
- [WMCTF2025-pdf2text-wp](../raw/web/WMCTF2025-pdf2text-wp.md)
- [D3CTF2019-ezupload-wp](../raw/web/D3CTF2019-ezupload-wp.md)
- [D3CTF2025-d3model-wp](../raw/web/D3CTF2025-d3model-wp.md)
