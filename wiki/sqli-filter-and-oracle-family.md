---
type: family
tags: [web, family, sqli, oracle, filter-bypass, blind-sqli, second-order]
skills: [ctf-web]
raw:
  - ../raw/web/sqli-filter-and-oracle.md
updated: 2026-07-06
---

# SQLi 过滤、输入面与 Oracle 技巧族

## 作用边界

本页是 Web SQLi family，负责从输入面、过滤器行为和反馈通道判断 SQLi 变体，再决定是直接提取数据，还是转向 XML/GraphQL 注入、上传/反序列化/RCE、认证边界或已知组件漏洞。

它不承担通用 Web 认证、浏览器侧外带或文件上传链路的目录职责；只有当核心证据仍是数据库查询语义、SQL 方言差异或 SQL oracle 时才留在本页。

## 识别信号

- 输入点位于 URL、body、JSON、Cookie、Header、上传元数据、二维码、EXIF、二阶存储字段或后台任务。
- 响应存在稳定差异：状态码、长度、排序、错误文本、时间、重定向、登录状态或后台副作用。
- 过滤器删除/替换关键字、转义引号、大小写归一、编码转换、WAF、ORM 或预处理逻辑异常。
- 数据库特性线索明显：MySQL truncation、SQLite timing、PostgreSQL cast/error、`information_schema` 被禁、processlist 可见。

## 最小证据

- 有一组正常请求和一组最小异常请求，并记录完整 HTTP transcript。
- 已确认差异由服务端查询语义导致，而不是缓存、CSRF、登录态、rate limit 或前端渲染。
- 已识别注入类型：回显、错误、布尔、时间、二阶、文件/元数据、编码绕过或 race/oracle。
- 如果使用工具失败，仍有手写 payload 能证明一个 bit、一个字符或一个状态差异。

## 分流流程

1. 固定 cookie、CSRF、账号状态和 baseline 响应。
2. 找到最小可控参数，用单字符 payload 验证引号、注释、编码和过滤行为。
3. 根据 oracle 选择提取策略：union/error/boolean/time/LIKE/race/second-order。
4. 把过滤绕过和提取逻辑分开脚本化，先验证一个字符，再扩展到完整 secret。
5. 如果 SQLi 只是链路一环，继续串 SSTI、文件写入、上传、session 覆盖或内部 API。

## SQLi 路线分流

| 变体 | 优先证据 | 下一跳页面 | 失败后 pivot |
|---|---|---|---|
| 引号/转义/编码绕过 | backslash、hex、Shift-JIS、full-width、双关键字 | [xml-command-and-graphql-injection.md](xml-command-and-graphql-injection.md) | 若 SQL 不通，检查 XML/GraphQL/模板/命令注入是否才是真入口。 |
| 布尔/时间盲注 | 长度、状态、延迟或排序稳定变化 | [web-first-pass-triage-and-chain-patterns.md](web-first-pass-triage-and-chain-patterns.md) | 若时间噪声大，改用状态差异、错误差异或二阶触发。 |
| 二阶 SQLi | 写入处无回显，后续页面/任务触发异常 | [auth-edge-cases-and-protocol-bypasses.md](auth-edge-cases-and-protocol-bypasses.md) | 若触发链不明，先枚举后台任务、profile、search、admin 页面。 |
| 元数据/文件型 SQLi | EXIF、CSV、QR、上传文件名进入查询 | [sqli-upload-deser-and-command-rce.md](sqli-upload-deser-and-command-rce.md) | 若数据库不可控，转上传解析、反序列化或文件包含。 |
| DB 特性替代 | `information_schema` 被禁、processlist、innodb stats | [known-cves-and-n-day-exploits.md](known-cves-and-n-day-exploits.md) | 若库版本有已知洞，验证 CVE/N-day 而不是继续盲猜 payload。 |
| SQLi to SSTI/RCE | SQL 结果进入模板、文件、session 或任务队列 | [ruby-php-upload-and-ssti-rce.md](ruby-php-upload-and-ssti-rce.md) | 若无法 RCE，退回 secret/token/session 读取。 |

## 常见陷阱

- sqlmap 没结果就放弃；CTF 题常把输入面藏在 header、metadata、二阶字段或编码层。
- 没固定 baseline 就比较响应差异，容易把缓存、登录态和 rate limit 当 oracle。
- 只堆 payload，不先理解过滤器是删除、替换、转义还是规范化。
- 盲注脚本没有重试和统计，时间型 oracle 很容易被网络噪声污染。
- 成功读库后停止；Web 题最终常需要把 SQLi 结果串到模板、上传、session 或内部接口。


## 关联技巧

- [web-first-pass-triage-and-chain-patterns.md](web-first-pass-triage-and-chain-patterns.md)
- [xml-command-and-graphql-injection.md](xml-command-and-graphql-injection.md)
- [sqli-upload-deser-and-command-rce.md](sqli-upload-deser-and-command-rce.md)
- [ruby-php-upload-and-ssti-rce.md](ruby-php-upload-and-ssti-rce.md)
- [php-lfi-ssti-ssrf-and-type-juggling.md](php-lfi-ssti-ssrf-and-type-juggling.md)
- [auth-edge-cases-and-protocol-bypasses.md](auth-edge-cases-and-protocol-bypasses.md)
- [parser-wrapper-and-legacy-ssrf-tricks.md](parser-wrapper-and-legacy-ssrf-tricks.md)
- [known-cves-and-n-day-exploits.md](known-cves-and-n-day-exploits.md)
- [web-tooling.md](web-tooling.md)

## 原始资料

- [sqli-filter-and-oracle.md](../raw/web/sqli-filter-and-oracle.md)
