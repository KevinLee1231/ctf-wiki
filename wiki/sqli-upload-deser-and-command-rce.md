---
type: family
tags: [web, family, sqli, upload, deserialization, command-injection, rce-chain]
skills: [ctf-web]
raw:
  - ../raw/web/sqli-upload-deser-and-command-rce.md
updated: 2026-06-11
---

# SQLi, Upload, Deserialization and Command RCE

## 作用边界

本页是 Web RCE 链路 family，覆盖 SQLi、上传、反序列化、命令包装器、备份泄露、文件读和自定义序列化等路线如何串到执行或敏感数据。它不再作为 technique，因为 raw 中的分支横跨数据库、文件系统、语言运行时、shell、PHP-FPM、反序列化和业务竞态。

与 [ruby-php-upload-and-ssti-rce.md](ruby-php-upload-and-ssti-rce.md) 的区别：本页更强调“已有一个入口后如何升级成执行链”；后者更强调语言/模板/上传本身的执行面识别。

## 链路分流

| 已有入口 | 优先升级路径 | 下一跳 |
|---|---|---|
| SQLi 可写文件、`INTO OUTFILE/DUMPFILE`、DNS SQLi | 写 WebShell、写配置、写 OPcache、读 secret 或构造二阶触发 | [sqli-filter-and-oracle-family.md](sqli-filter-and-oracle-family.md)、[path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md) |
| 上传图片、BMP/PNG/PHP polyglot、双扩展、`.phar` | 同时满足内容、后缀、MIME、服务器解析和访问路径 | [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md)、[ruby-php-upload-and-ssti-rce.md](ruby-php-upload-and-ssti-rce.md) |
| Java/Pickle/PHP 反序列化 | 确认入口、类加载/gadget、过滤器、编码层和触发时机 | [php-java-python-deserialization.md](php-java-python-deserialization.md) |
| 命令包装器、tar/wget/date/shell 拼接 | 文件名、参数、换行、brace expansion、环境变量或错误输出是否可控 | [parser-wrapper-and-legacy-ssrf-tricks.md](parser-wrapper-and-legacy-ssrf-tricks.md) |
| 备份文件、swap、源码泄露、任意文件读 | 先读源码、配置、密钥、路由和部署信息，再回到具体漏洞链 | [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md) |
| 自定义 serializer、长度截断、整数溢出 | 解析器读取字段顺序是否可被长度/类型混淆改写 | [json-duplicate-key-hmac-parser-differential.md](json-duplicate-key-hmac-parser-differential.md)、[auth-bypass-cookies-and-hidden-routes.md](auth-bypass-cookies-and-hidden-routes.md) |
| 业务竞态或 TOCTOU | 状态检查和状态使用是否跨请求、跨锁或跨队列 | [race-condition-and-concurrency-exploits.md](race-condition-and-concurrency-exploits.md) |

## 合并与拆分结论

- 保留为 family：它的价值是把多个入口统一映射到“升级为文件读、secret、WebShell、反序列化或命令执行”的链路选择。
- 不合并进 SQLi family：SQLi 只是其中一个入口；本页关注 SQLi 之后的横向升级。
- 不合并进上传 family：本页还覆盖命令包装器、serializer 和反序列化链。
- 当前不拆成独立 technique：大多数分支已有更具体页面承接，raw 主要作为短案例集合和链路速查。

## 常见误判

- SQLi 已经能读库就停止，没有继续找文件写、配置密钥、模板路径或 session secret。
- 上传 payload 满足内容检查，却忽略最终由谁解析：Nginx、Apache、PHP-FPM、框架静态目录或对象存储。
- 反序列化只测试 gadget，不确认反序列化发生的请求路径和触发对象。
- 命令注入只看 `;`，忽略空格过滤、参数注入、换行、glob、brace expansion 和错误输出外带。

## 关联页面

- [web-first-pass-triage-and-chain-patterns.md](web-first-pass-triage-and-chain-patterns.md)
- [sqli-filter-and-oracle-family.md](sqli-filter-and-oracle-family.md)
- [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md)
- [ruby-php-upload-and-ssti-rce.md](ruby-php-upload-and-ssti-rce.md)
- [php-java-python-deserialization.md](php-java-python-deserialization.md)
- [parser-wrapper-and-legacy-ssrf-tricks.md](parser-wrapper-and-legacy-ssrf-tricks.md)

## 原始资料

- [sqli-upload-deser-and-command-rce.md](../raw/web/sqli-upload-deser-and-command-rce.md)
