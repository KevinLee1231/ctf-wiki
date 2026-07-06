---
type: family
tags: [web, family, rce, upload, ssti, code-injection]
skills: [ctf-web]
raw:
  - ../raw/web/ruby-php-upload-and-ssti-rce.md
updated: 2026-07-06
---

# Ruby, PHP, Upload and SSTI RCE

## 作用边界

本页是 Web RCE family：用于判断输入是否已经进入语言求值、模板求值、文件上传解析、反序列化、命令拼接或脚本执行链。它不再作为一个单点 technique，因为 raw 覆盖 Ruby、PHP、Perl、JS、LaTeX、Prolog、Thymeleaf、f-string、上传和 API 注入等多种执行面。

与 [php-lfi-ssti-ssrf-and-type-juggling.md](php-lfi-ssti-ssrf-and-type-juggling.md) 的区别：本页的目标是确认并利用执行点；后者更多负责判断服务端解析差异是否能通向读文件、内部访问或认证绕过。

边界在于“执行面确认”：如果只有 LFI/SSRF/XXE/弱比较/格式化器差异，还没有证明代码、模板或命令能执行，先回到 [php-lfi-ssti-ssrf-and-type-juggling.md](php-lfi-ssti-ssrf-and-type-juggling.md) 或 [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md)；本页只承接已经能定位执行点或能把上传/模板/反序列化推进到执行的链路。

## 识别信号

- 输入已经进入 `eval`、模板求值、语言对象遍历、命令拼接、WebShell、上传解析、反序列化 gadget 或脚本执行点。
- 响应能体现表达式执行、模板报错、命令输出、文件写入、外连、延时、异常栈或可控副作用。
- 上传链同时具备内容可控、落点可控、解析规则可控或可访问路径可控中的至少两项。
- 过滤器阻断常规 payload，但执行上下文仍可通过拼接、编码、变量函数、header、文件名或环境变量触达。

## 最小证据

- 明确执行器类型：Ruby/PHP/Perl/JS/LaTeX/模板引擎/shell/反序列化/上传解析。
- 保存一个不会破坏环境的最小 side effect，例如算术回显、时间延迟、写临时文件、DNS 请求或读取无敏感固定文件。
- 对上传链，要同时记录文件名、内容、MIME、落点、访问 URL、服务器映射和最终解释器。
- 对模板/语言 eval，要记录当前上下文、可用对象、过滤位置和 payload 如何闭合原语法。

## 变体路由

| 信号 | 先确认什么 | 下一跳 |
|---|---|---|
| Ruby `instance_eval`、`ObjectSpace`、private method、blocklist | 是否能闭合当前表达式并执行 Ruby 语句；是否能扫描堆或绕 private visibility | 本页 raw；需要绕沙箱时转 [pyjails.md](pyjails.md) |
| PHP `preg_replace /e`、`assert()`、反引号、`eval()`、`create_function` | 用户输入是否进入字符串求值；长度/字符过滤是否能用 header、变量函数或拼接绕过 | [php-lfi-ssti-ssrf-and-type-juggling.md](php-lfi-ssti-ssrf-and-type-juggling.md) |
| Java/Thymeleaf SpEL、Jinja2、Mako、Twig、EJS、Go template | 模板上下文、对象图、WAF 过滤和可达文件/命令 API | [php-lfi-ssti-ssrf-and-type-juggling.md](php-lfi-ssti-ssrf-and-type-juggling.md) |
| 上传 ZIP、图片 polyglot、`.htaccess`、日志投毒、WebShell | 文件内容、扩展名、MIME、解压路径和服务器解析规则是否同时可控 | [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md) |
| Cookie/POST/body 中的 Pickle、Java、PHP object | 反序列化入口、gadget、过滤器和触发时机 | [php-java-python-deserialization.md](php-java-python-deserialization.md) |
| `open()`、shell 拼接、tar/wget/date 等命令包装 | 空格、引号、换行、brace expansion、文件名参数是否能改变命令语义 | [parser-wrapper-and-legacy-ssrf-tricks.md](parser-wrapper-and-legacy-ssrf-tricks.md) |
| API filter/query 参数能改变字段或查询结构 | 是数据库/ORM 注入、mass assignment，还是业务级权限绕过 | [auth-edge-cases-and-protocol-bypasses.md](auth-edge-cases-and-protocol-bypasses.md)、[sqli-filter-and-oracle-family.md](sqli-filter-and-oracle-family.md) |

## 合并与拆分结论

- 保留为 family：这些案例共享“输入进入可执行解释层”的判断，但具体语言和触发面差异很大，不适合伪装成一个 technique。
- 不与上传/路径 family 合并：上传链的关键常常是文件落点和解析规则；本页只在上传最终形成执行时承接。
- 不拆出每种语言小页：当前 raw 多数是短案例速查，单独拆页会形成低信息密度页面；有多篇 WP 支撑时再拆具体 technique。

## 失败后转向

- 能读文件但不能执行：退回 [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md) 或 LFI/SSTI family，先找 secret、源码和配置。
- RCE payload 被过滤：先定位过滤发生在输入、模板、shell 还是 WAF 层，再决定用编码、拼接、环境变量、文件名或 header。
- 反序列化 gadget 不触发：检查对象入口、类加载、依赖版本和 magic method；不要只换 ysoserial 链。
- 上传成功但不能访问：确认 Web root、随机目录、扩展映射、`.htaccess` 生效范围和解压后路径。

## 常见误判

- 只看到模板报错就认定 RCE，没有确认对象图能到达文件、命令或可验证副作用。
- 上传成功后忽略 Web 服务器和语言运行时的解析边界，把“文件落地”误当“代码执行”。
- 命令拼接题只测试分号，没有检查参数注入、换行、glob、环境变量和错误输出外带。
- 反序列化题只套 gadget，不确认入口是否真的反序列化了攻击者对象。

## 关联页面

- [web-first-pass-triage-and-chain-patterns.md](web-first-pass-triage-and-chain-patterns.md)
- [php-lfi-ssti-ssrf-and-type-juggling.md](php-lfi-ssti-ssrf-and-type-juggling.md)
- [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md)
- [php-java-python-deserialization.md](php-java-python-deserialization.md)
- [parser-wrapper-and-legacy-ssrf-tricks.md](parser-wrapper-and-legacy-ssrf-tricks.md)
- [sqli-filter-and-oracle-family.md](sqli-filter-and-oracle-family.md)

## 原始资料

- [ruby-php-upload-and-ssti-rce.md](../raw/web/ruby-php-upload-and-ssti-rce.md)
