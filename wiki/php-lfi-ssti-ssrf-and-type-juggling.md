---
type: family
tags: [web, family, php, lfi, ssti, ssrf, parser-confusion]
skills: [ctf-web]
raw:
  - ../raw/web/php-lfi-ssti-ssrf-and-type-juggling.md
updated: 2026-06-11
---

# PHP, LFI, SSTI, SSRF and Type Juggling

## 作用边界

本页是 Web 服务端解析差异的 family，不再作为单一 technique 使用。它覆盖的共同信号是：用户输入进入 PHP/模板/XML/URL fetch/格式化器/弱比较等服务端解释层，校验与执行层看到的值不一致，最终形成认证绕过、源码读取、内部访问或 RCE。

不应把这里当成 payload 字典。先判断输入进入了哪一种解释器，再跳到具体 technique 或相邻 family。

## 首轮变体判断

| 信号 | 优先判断 | 下一跳 |
|---|---|---|
| PHP `==`、`strcmp`、`hash_hmac`、数组参数、`0e...` hash | 弱类型或函数错误返回值是否进入认证/签名判断 | [auth-bypass-cookies-and-hidden-routes.md](auth-bypass-cookies-and-hidden-routes.md)、[auth-edge-cases-and-protocol-bypasses.md](auth-edge-cases-and-protocol-bypasses.md) |
| `include` / `require` / `php://filter` / `data:` / 日志包含 | 先证明可读源码，再判断是否可写入或远程包含 | [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md)、[ruby-php-upload-and-ssti-rce.md](ruby-php-upload-and-ssti-rce.md) |
| `{{7*7}}`、模板报错、Jinja2/Go/EJS/ERB/Mako/Twig/Smarty/Vue | 确认模板引擎、表达式上下文、过滤器和对象可达性 | [ruby-php-upload-and-ssti-rce.md](ruby-php-upload-and-ssti-rce.md) |
| URL fetch、Host header、curl redirect、DNS rebinding | 校验点和实际请求点是否使用不同解析结果 | [artifact-trust-ssrf-to-node-require-rce.md](artifact-trust-ssrf-to-node-require-rce.md)、[polyglot-url-tricks-and-ssrf-leaks.md](polyglot-url-tricks-and-ssrf-leaks.md)、[parser-wrapper-and-legacy-ssrf-tricks.md](parser-wrapper-and-legacy-ssrf-tricks.md) |
| XML 上传、DOCX/Office XML、外部 DTD | 解析器是否允许外部实体、网络请求或本地文件读取 | [xml-command-and-graphql-injection.md](xml-command-and-graphql-injection.md) |
| SQL、Python `str.format`、格式化表达式混入 | 如果核心是数据库 oracle，转 SQLi；如果核心是对象属性遍历，按语言执行链处理 | [sqli-filter-and-oracle-family.md](sqli-filter-and-oracle-family.md)、[ruby-php-upload-and-ssti-rce.md](ruby-php-upload-and-ssti-rce.md) |

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

## 原始资料

- [php-lfi-ssti-ssrf-and-type-juggling.md](../raw/web/php-lfi-ssti-ssrf-and-type-juggling.md)
