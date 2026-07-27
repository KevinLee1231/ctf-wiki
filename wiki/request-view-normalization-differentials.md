---
type: technique
tags: [web, request-smuggling, parser-differential, normalization, technique]
skills: [ctf-web]
raw:
  - ../raw/web/UMDCTF2024-http-fanatics-wp.md
  - ../raw/web/UMDCTF2024-donations-2-wp.md
  - ../raw/web/UMDCTF2025-steve-le-poisson-wp.md
  - ../raw/web/UMDCTF2025-tiktok-ban-wp.md
  - ../raw/web/UMDCTF2026-offknock-wp.md
  - ../raw/web/SekaiCTF2026-migurimental-wp.md
  - ../raw/web/D3CTF2024-d3pythonhttp-wp.md
  - ../raw/web/SUCTF2026-wmsWP.md
updated: 2026-07-27
---

# Request View and Normalization Differentials

## 适用场景

同一请求依次经过代理、协议转换器、中间件、路由、授权层和业务处理器，各层读取的可能是原始字节、规范化路径、首个/末个重复字段、合并 header、重写后的内部参数或重新解析的协议对象。若安全检查与最终消费不在同一表示上，就会出现授权、路由或消息边界绕过。

## 识别信号

- 重复 query、Cookie、header 或 JSON key 在不同组件中取首值、末值、数组或合并字符串。
- HTTP/2/3 被降级到 HTTP/1.1，正文边界同时受 stream、Content-Length 或 Transfer-Encoding 影响。
- ACL 检查原始 path，路由层随后做 decode、大小写折叠、rewrite、asset prefix 或内部参数规范化。
- 过滤器扫描原始 DNS/URL 字节，后端解析器却支持压缩指针、畸形编码或另一套 canonicalization。

## 最小证据

- 保存请求在每一层的可见表示，至少覆盖安全检查点和最终 sink。
- 用成对请求证明只有表示差异变化，业务结果或授权结果随之改变。
- 明确重复字段采用首值、末值、合并还是拒绝，以及规范化发生的准确顺序。
- 协议转换场景要画出每一层的消息边界，不能只看抓包中的外层请求。

## 解法骨架

1. 建立“原始字节 -> 代理 -> 框架 -> 中间件 -> 路由 -> sink”的解释链。
2. 对安全敏感字段逐层记录名称、值、重复项、编码和 canonical path。
3. 选择一个最小差异，使检查层看到允许值而消费层看到攻击值。
4. 分别验证授权绕过、二段请求、SQL/反序列化 sink 或内部路由确实被触发。
5. 修复判断应落在统一解析后的最终对象，并拒绝重复或歧义表示。

## 关键变体

| 差异类型 | 首轮测试 |
|---|---|
| 重复字段 | 两个相反值，比较原始日志、中间件对象和业务读取结果。 |
| 路径规范化 | 组合 decode、分号参数、大小写、斜杠、rewrite 和内部数据路由。 |
| 协议转换 | 检查上游 stream 结束与下游 CL/TE/chunked 是否产生不同边界。 |
| 线缆格式 | 比较原始 DNS/URL 字节与完整解析后的名称、标签和指针落点。 |
| 动态驱动参数 | 检查白名单只约束前缀，而 JDBC/URL 解析器继续解释危险选项。 |

## 常见陷阱

- 只在单一框架实例中测试，没有复现真实代理和协议降级链。
- 用正则扫描原始 body/header 后认为字段已校验，忽略后续重新解析。
- 发现解析差异就直接称为 request smuggling，没有证明第二个边界或授权结果。
- 修复仅新增黑名单字符串，没有统一解析并拒绝歧义输入。

## 关联技巧

- [web-first-pass-triage-and-chain-patterns.md](web-first-pass-triage-and-chain-patterns.md)
- [json-duplicate-key-hmac-parser-differential.md](json-duplicate-key-hmac-parser-differential.md)
- [path-confusion-to-signed-internal-request-chain.md](path-confusion-to-signed-internal-request-chain.md)
- [parser-wrapper-and-legacy-ssrf-tricks.md](parser-wrapper-and-legacy-ssrf-tricks.md)
- [php-java-python-deserialization.md](php-java-python-deserialization.md)

## 来自 WP 的案例索引

本节只保留可复用识别信号，不替代原始题解正文。

| Raw WP | 可复用联系 |
|---|---|
| [0xGame2025-week3-这真的是文件上传-wp](../raw/web/0xGame2025-week3-这真的是文件上传-wp.md) | 本题不是单纯的文件上传，而是“路径规范化差异 + 任意文件覆盖 + EJS 模板执行”的组合漏洞。安全检查基于规范化前的字符串，而文件系统操作使用规范化后的路径，导致 `/.` 绕过；上传接口又允许覆盖可执行模板，最终把写文件能力升级为服务端代码执行。修复时应先解析并规范化路径，再校验其真实落点是否位于专用上传目录，同时禁止覆盖模板等可执行资源。 |
| [ACTF2023-mygos-live-wp](../raw/web/ACTF2023-mygos-live-wp.md) | JavaScript `parseInt` 接受数字前缀，Bash 将制表符解释为空白，`?` 又可代替被过滤的 `*` 完成 glob。应避免 `bash -c`，严格校验完整端口字符串，并用独立管道或临时文件隔离每次请求的输出。 |
| [MoeCTF2022-支付系统-wp](../raw/web/MoeCTF2022-支付系统-wp.md) | 对任何“多字段拼接后签名”的协议，都应先写出验签前的确切字节串，再检查字段边界是否唯一。安全做法是使用无歧义的规范序列化，例如固定宽度、长度前缀或带类型的规范 JSON，并在服务端从数据库读取不可变交易字段后自行重建消息。验签通过后也不应把回调表单中的任意字段直接写回原交易，否则签名覆盖的字节语义与业务层解析语义一旦不一致，就会出现本题这种签名重用。 |
| [UMDCTF2022-customer-support-wp](../raw/web/UMDCTF2022-customer-support-wp.md) | 本题不是 DNS rebinding，而是 URL 解析结果为空时的危险默认值。过滤发生在原始文本层，实际请求目标却由解析后的字段和内部环境变量重新拼接，两者语义不一致。修复时应使用 WHATWG `URL` 严格解析，解析失败直接拒绝，并在连接前后都验证最终 IP、端口和重定向目标；更不能把内部服务地址作为空主机的回退值。 |
| [UMDCTF2023-homework-render-wp](../raw/web/UMDCTF2023-homework-render-wp.md) | 决定性漏洞不是单独的 LaTeX 注入，而是两层框架对重复参数采用不同取值语义；代理、框架和后端之间若没有把多值参数规范化，前置校验就可能检查一份数据、后端使用另一份数据。 |
| [UMDCTF2025-a-minecraft-movie-wp](../raw/web/UMDCTF2025-a-minecraft-movie-wp.md) | 净化后保留的命名元素可通过 DOM named properties 改变全局变量解析，再借未 URL 编码的模板字符串注入重复参数。客户端应使用 `URLSearchParams`，后端应拒绝重复关键参数并独立验证 session number。 |
| [WMCTF2022-easyjeecg-wp](../raw/web/WMCTF2022-easyjeecg-wp.md) | 本题核心是两层解析差异：Spring 拦截器按字符串白名单判断 `toLogin.do`，Tomcat 按路径参数语义处理 `;`，Nginx 又只按原始 URI 正则拦截 `/upload/*.jsp`。把三者串起来即可未授权上传并执行 JSP。 |
| [WMCTF2023-anyfileread-wp](../raw/web/WMCTF2023-anyfileread-wp.md) | 利用路径归一化差异绕过任意文件读取过滤；接口允许传入文件路径，过滤只针对字面量 `../` 或固定目录前缀时，应测试重复斜杠、URL 编码、双重编码和混合分隔符。 |
| [WMCTF2024-passwdstealer-wp](../raw/web/WMCTF2024-passwdstealer-wp.md) | 识别点：SpringBoot + Tomcat 9.0.43 + multipart 上传处理；核心条件：超时后不返回 500、连接保持 keep-alive、后续请求能回显。 |

## 原始资料

- [UMDCTF2024-http-fanatics-wp.md](../raw/web/UMDCTF2024-http-fanatics-wp.md)
- [UMDCTF2024-donations-2-wp.md](../raw/web/UMDCTF2024-donations-2-wp.md)
- [UMDCTF2025-steve-le-poisson-wp.md](../raw/web/UMDCTF2025-steve-le-poisson-wp.md)
- [UMDCTF2025-tiktok-ban-wp.md](../raw/web/UMDCTF2025-tiktok-ban-wp.md)
- [UMDCTF2026-offknock-wp.md](../raw/web/UMDCTF2026-offknock-wp.md)
- [SekaiCTF2026-migurimental-wp.md](../raw/web/SekaiCTF2026-migurimental-wp.md)
- [D3CTF2024-d3pythonhttp-wp.md](../raw/web/D3CTF2024-d3pythonhttp-wp.md)
- [SUCTF2026-wmsWP.md](../raw/web/SUCTF2026-wmsWP.md)
