---
type: family
tags: [web, family, php, lfi, ssti, ssrf, parser-confusion]
skills: [ctf-web]
raw:
  - ../raw/web/php-lfi-ssti-ssrf-and-type-juggling.md
updated: 2026-07-27
---

# PHP, LFI, SSTI, SSRF and Type Juggling

## 作用边界

本页是 Web 服务端解析差异的 family，不再作为单一 technique 使用。它覆盖的共同信号是：用户输入进入 PHP/模板/XML/URL fetch/格式化器/弱比较等服务端解释层，校验与执行层看到的值不一致，最终形成认证绕过、源码读取、内部访问或 RCE。

不应把这里当成 payload 字典。先判断输入进入了哪一种解释器，再跳到具体 technique 或相邻 family。

边界在于“解释层差异判断”：如果当前证据只证明能读源码、访问内网、改变解析结果或绕过认证，先留在本页或转资源/SSRF 页面；如果已经确认输入进入 `eval`、模板求值、命令拼接、WebShell 或可执行上传，转 [ruby-php-upload-and-ssti-rce.md](ruby-php-upload-and-ssti-rce.md) 处理 RCE 落地。

## 识别信号

- 输入进入 PHP 弱类型比较、文件包含、模板表达式、URL fetch、XML parser、格式化器或其它服务端解释器。
- 校验层和执行层对同一值的解析结果不同：类型、路径、host、协议、模板对象、XML 实体或格式化字段发生变化。
- 响应暴露源码、内部响应、模板报错、XML 实体错误、过滤器行为、认证状态或可控 include 路径。
- 题目还没证明代码执行，但已经能证明服务端解释层可被影响。

## 最小证据

- 保存能复现差异的最小输入，并标出它进入的是 PHP 比较、文件包含、模板、URL fetch、XML parser 还是格式化器。
- 记录校验层与执行层看到的关键值：类型转换结果、归一化路径、最终 host、模板上下文、XML 实体展开或格式化字段。
- 若目标是源码/secret 泄露，至少拿到一处可控文件读取、源码片段、配置值、错误回显或内部响应。
- 若准备转 RCE family，必须先证明输入已到达可执行解释层或可写入可执行位置，不能只凭报错或黑盒猜测跳转。

## 首轮变体判断

| 信号 | 优先判断 | 下一跳 |
|---|---|---|
| PHP `==`、`strcmp`、`hash_hmac`、数组参数、`0e...` hash | 弱类型或函数错误返回值是否进入认证/签名判断 | [auth-bypass-cookies-and-hidden-routes.md](auth-bypass-cookies-and-hidden-routes.md)、[auth-edge-cases-and-protocol-bypasses.md](auth-edge-cases-and-protocol-bypasses.md) |
| `include` / `require` / `php://filter` / `data:` / 日志包含 | 先证明可读源码，再判断是否可写入或远程包含 | [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md)、[ruby-php-upload-and-ssti-rce.md](ruby-php-upload-and-ssti-rce.md) |
| `{{7*7}}`、模板报错、Jinja2/Go/EJS/ERB/Mako/Twig/Smarty/Vue | 确认模板引擎、表达式上下文、过滤器和对象可达性 | [ruby-php-upload-and-ssti-rce.md](ruby-php-upload-and-ssti-rce.md) |
| URL fetch、Host header、curl redirect、DNS rebinding | 校验点和实际请求点是否使用不同解析结果 | [artifact-trust-ssrf-to-node-require-rce.md](artifact-trust-ssrf-to-node-require-rce.md)、[polyglot-url-tricks-and-ssrf-leaks.md](polyglot-url-tricks-and-ssrf-leaks.md)、[parser-wrapper-and-legacy-ssrf-tricks.md](parser-wrapper-and-legacy-ssrf-tricks.md) |
| XML 上传、DOCX/Office XML、外部 DTD | 解析器是否允许外部实体、网络请求或本地文件读取 | [xml-command-and-graphql-injection.md](xml-command-and-graphql-injection.md) |
| SQL、Python `str.format`、格式化表达式混入 | 如果核心是数据库 oracle，转 SQLi；如果核心是对象属性遍历，按语言执行链处理 | [sqli-filter-and-oracle-family.md](sqli-filter-and-oracle-family.md)、[ruby-php-upload-and-ssti-rce.md](ruby-php-upload-and-ssti-rce.md) |

## Technique 下一跳

| 首轮判断 | 具体 technique |
|---|---|
| 可控路径进入 include/read，wrapper 或 session/log 文件可利用 | [lfi-wrapper-and-session-file-inclusion.md](lfi-wrapper-and-session-file-inclusion.md) |
| URL 校验器与请求端、wrapper 或重定向后的终点不同 | [url-parser-wrapper-and-ssrf-filter-differential.md](url-parser-wrapper-and-ssrf-filter-differential.md) |
| 模板/表达式/命令出现可验证的二次解释 | [server-side-expression-and-command-context-escape.md](server-side-expression-and-command-context-escape.md) |

## 合并与拆分结论

- 保留为 family：raw 中的 PHP 弱类型、LFI、SSTI、SSRF、XXE 和格式化器案例共享“服务端解释层差异”这一首轮判断，单独拆成多个薄页会重复 raw 速查内容。
- 不合并进 Web 首轮页：这里的判断已经进入服务端解释器层，比 [web-first-pass-triage-and-chain-patterns.md](web-first-pass-triage-and-chain-patterns.md) 更具体。
- 不合并进 RCE family：本页不一定走到代码执行，很多路线只需要源码、secret、内部响应或认证绕过。
- 后续若某个分支积累多个独立 WP 且有稳定最小证据，可拆成具体 technique，例如 `php-wrapper-file-inclusion.md` 或 `ssti-object-traversal-rce.md`。

## 常见误判

- 看到 `php://filter` 就直接打 RCE；多数情况下第一价值是源码泄露和 secret recovery。
- SSTI payload 只复制引擎通用链，没有先确认模板上下文、字符串过滤和对象可达性。
- SSRF payload 只验证外网回连，没有确认内网目标、跳转处理和响应是否可回显。
- XXE 在上传容器格式中触发时，真正入口可能是 DOCX/SVG/PDF 解析器，而不是普通 XML API。

## 关联页面

- [web-first-pass-triage-and-chain-patterns.md](web-first-pass-triage-and-chain-patterns.md)
- [ruby-php-upload-and-ssti-rce.md](ruby-php-upload-and-ssti-rce.md)
- [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md)
- [sqli-filter-and-oracle-family.md](sqli-filter-and-oracle-family.md)
- [artifact-trust-ssrf-to-node-require-rce.md](artifact-trust-ssrf-to-node-require-rce.md)
- [xml-command-and-graphql-injection.md](xml-command-and-graphql-injection.md)

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [ACTF2026-tinj-wp](../raw/web/ACTF2026-tinj-wp.md) | JPHP/Jetty 字符字节转换绕过路由后进入 eval、JAR/classloader 和内存马链，先确认运行时边界。 |
| [TAMUctf2022-Serial-Killer-wp](../raw/web/TAMUctf2022-Serial-Killer-wp.md) | Cookie 解码后是 PHP 序列化对象，`file` 属性进入 include；关键是同步长度字段并利用 URL 解码顺序绕过 `../` 过滤。 |
| [D3CTF2019-fake-onelinephp-wp](../raw/web/D3CTF2019-fake-onelinephp-wp.md) | Windows PHP 文件包含可走 WebDAV/SMB 远程包含，再从 git 和内网凭据继续 pivot。 |
| [D3CTF2021-happy-valentines-day-wp](../raw/web/D3CTF2021-happy-valentines-day-wp.md) | 模板/SSTI 语义是主入口，先确认 Java/模板上下文和可达命令执行对象。 |
| [VNCTF2026-signin-wp](../raw/web/VNCTF2026-signin-wp.md) | PHP `include` 可控且过滤 `php://filter` 常见写法，先用短 `data:,<?=...` 伪协议构造执行内容。 |
| [0xGame2021-week2-find-my-secret-wp](../raw/web/0xGame2021-week2-find-my-secret-wp.md) | 利用链为“数组触发类型错误 → 秘密 HMAC 密钥退化为空 → 本地计算校验值 → 动态调用 `create_function()` → 闭合函数体执行代码”。该链依赖旧 PHP 行为；现代 PHP 已移除 `create_function()`，并会对许多错误类型参数抛出 `TypeError`。根本修复仍是严格校验参数类型、拒绝用户控制可调用函数名，并使用固定的代码路径。 |
| [0xGame2024-week1-ez-unser-wp](../raw/web/0xGame2024-week1-ez-unser-wp.md) | 构造 PHP POP 链，分别获得任意内容写入和同目录文件重命名两个原语，再组合为可执行 PHP 文件；入口存在无约束 `unserialize()`，并且类中出现 `__wakeup`、`__toString`、`__get`、`__debugInfo`、`__call` 等可连续触发的魔术方法。 |
| [0xGame2025-week2-我只想要你的PNG-wp](../raw/web/0xGame2025-week2-我只想要你的PNG-wp.md) | 核心问题不是普通的“上传 PHP 文件”，而是未经转义的上传文件名被写入 PHP 源文件；`accept="image/*"`、客户端 MIME 类型和文件名后缀都不能证明文件可信；服务端还应校验内容，并把用户数据写入不可执行目录。 |
| [0xGame2025-week4-绳网委托Bottle版-wp](../raw/web/0xGame2025-week4-绳网委托Bottle版-wp.md) | 本题的关键不是绕过双花括号，而是识别同一模板引擎提供的另一套语法。黑名单只覆盖常见 payload 形式，没有改变“用户输入被当作模板源码再次编译”的根本问题。修复时应只把固定模板交给 `template()`，用户内容必须作为数据变量传入并按上下文转义，不能依靠删除少量字符或关键字建立安全边界。 |
| [ACTF2025-not-so-web-1-wp](../raw/web/ACTF2025-not-so-web-1-wp.md) | AES-CBC 只提供保密性，不提供完整性。IV 与首块明文直接异或，客户端可控 IV 加上已知 JSON 前缀就能定向修改身份字段。修复时应使用 AES-GCM、ChaCha20-Poly1305 等 AEAD，或至少对 `IV \|\| ciphertext` 进行先验 MAC 校验；管理员页面还必须把用户输入作为模板数据传入固定模板，不能先拼接再交给 `render_template_string()`。 |
| [MoeCTF2025-22-血海核心-千年手段-wp](../raw/web/MoeCTF2025-22-血海核心-千年手段-wp.md) | 本题包含两层边界：先把无回显 SSTI 转化为静态文件回显，再利用自定义 SUID 包装器跨越文件权限。看到陌生 SUID 程序时不能只试命令名，应检查其参数解析和 `exec*()` 调用；这里的固定开关 `--HDdss` 正是提权入口。无回显场景下，优先寻找应用自身可读取的写入位置，比部署复杂内存马更短、更稳定。 |
| [MoeCTF2025-附加挑战-wp](../raw/web/MoeCTF2025-附加挑战-wp.md) | 本题把自定义反射 gadget 与经典 `TemplatesImpl` 动态类加载连接起来，再用 Tomcat 请求线程中的对象解决无回显问题。关键细节有三个：类型转换发生在 `readObject()` 之后，无法阻止 gadget；恶意类必须继承 `AbstractTranslet`；ThreadLocal 回显需要先开启开关、再在下一次请求中读取当前对象。相比注册长期驻留的拦截器，一次性回显链足以完成题目，也更容易复现和审计。 |
| [MoeCTF2025-webshell-revenge-wp](../raw/web/MoeCTF2025-webshell-revenge-wp.md) | 本题把 PHP 语法、URL 表单解码、multipart 临时文件和 shell glob 串成了一条跨层利用链。遇到极短且禁止变量的 `eval()` 时，不一定要继续在 PHP 内构造函数名；若同一请求还能制造服务器端临时文件，就可以让极短载荷只负责定位和执行文件，把复杂命令移到不受过滤的上传内容中。 |
| [WMCTF2020-checkin1-2-wp](../raw/web/WMCTF2020-checkin1-2-wp.md) | 利用 `php://filter` 在 `file_put_contents` 写入阶段处理内容，结合二次 URL 解码、未过滤的 `zlib/string` 过滤器或 PHP 7.0.33 `string.strip_tags` 临时文件行为绕过 `exit()` 前缀。 |
| [WMCTF2020-login-me-again-and-again-wp](../raw/web/WMCTF2020-login-me-again-and-again-wp.md) | Host 头绕过反向代理、分字段 SSTI 拼接、SSTI 动态注册 Flask 路由、reGeorg HTTP 隧道访问内网 MySQL；代理源码注释暴露内网 host、路由 method 拼写异常、模板过滤按字段分别处理、flag 不在当前容器且存在内网服务依赖时，应考虑从 SSTI 扩展到内网隧道。 |

## 原始资料

- [php-lfi-ssti-ssrf-and-type-juggling.md](../raw/web/php-lfi-ssti-ssrf-and-type-juggling.md)
