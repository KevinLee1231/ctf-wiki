---
type: technique
tags: [web, technique]
skills: [ctf-web]
raw:
  - ../raw/web/xml-command-and-graphql-injection.md
updated: 2026-05-21
---

# XML, Command and GraphQL Injection

## 适用场景

HTTP 行为、认证授权、服务端解析差异、文件/路径处理、模板或浏览器执行模型是主要障碍。

本页不是 raw 的目录页；它把原始资料中的案例压缩成可迁移的判断信号、最小证据和解题骨架。

## 识别信号

- 请求、路由、Cookie、Header、body 或服务端状态出现可控差异。
- 源码或响应能定位到中间件、鉴权、解析器、模板、上传或内部 API。
- 能用最小 HTTP payload 复现行为变化。
- 题面或 raw 线索能落到这些关键词之一：XXE (XML External Entity)、Basic XXE、OOB XXE with External DTD、XXE via DOCX/Office XML Upload (School CTF 2016)、SVG XXE via svglib to PNG Pipeline (P.W.N. CTF 2018)、XML Injection via X-Forwarded-For Header (Pwn2Win 2016)、PHP Variable Variables ($$var) Abuse (bugsbunny 2017)、PHP uniqid() Predictable Filename (EKOPARTY 2017)。

## 最小证据

- 已完成主方向判断，并确认本页技巧比相邻技巧更能解释当前证据。
- 至少有一个可复现输入、输出、文件结构、数学关系、协议行为或运行时状态。
- 能指出 raw 案例中哪一个变体与当前题最接近，以及不同点在哪里。

## 解法骨架

1. 固定登录态和正常业务流。
2. 构造最小请求验证一个主假设。
3. 把单点漏洞串成 secret/token/internal API/RCE。
4. 写可重放 solver 并记录关键响应。

## 关键变体

| 变体 | 复用重点 |
|---|---|
| XXE (XML External Entity) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Basic XXE | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| OOB XXE with External DTD | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| XXE via DOCX/Office XML Upload (School CTF 2016) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| SVG XXE via svglib to PNG Pipeline (P.W.N. CTF 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| XML Injection via X-Forwarded-For Header (Pwn2Win 2016) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| PHP Variable Variables ($$var) Abuse (bugsbunny 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| PHP uniqid() Predictable Filename (EKOPARTY 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Sequential Regex Replacement Bypass (Tokyo Westerns 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Command Injection | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Newline Bypass | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Incomplete Blocklist Bypass | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Sendmail Parameter Injection via CGI (SECCON 2015) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Multi-Barcode Concatenation to Shell Injection (BSidesSF 2024) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Git CLI Newline Injection via URL Path (BSidesSF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Command Injection 速查 | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| GraphQL Injection and Exploitation (Hack.lu CTF 2020, HeroCTF v5) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Introspection and Schema Discovery | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Query Batching and Aliasing for Rate Limit Bypass | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| String Interpolation Injection | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| XXE 速查 | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |

## 常见陷阱

- 只按关键词跳页，没有先构造最小证据。
- 照搬 raw 中的一次性 payload，没有检查当前题的边界条件。
- 忽略相邻技巧之间的 pivot，导致在错误方向上继续投入时间。

## 关联技巧

- [sqli-filter-and-oracle-family.md](sqli-filter-and-oracle-family.md)
- [artifact-trust-ssrf-to-node-require-rce.md](artifact-trust-ssrf-to-node-require-rce.md)
- [auth-bypass-cookies-and-hidden-routes.md](auth-bypass-cookies-and-hidden-routes.md)
- [auth-edge-cases-and-protocol-bypasses.md](auth-edge-cases-and-protocol-bypasses.md)
- [auth-jwt.md](auth-jwt.md)
- [csp-xsleak-and-browser-exfiltration.md](csp-xsleak-and-browser-exfiltration.md)

## 原始资料

- [xml-command-and-graphql-injection.md](../raw/web/xml-command-and-graphql-injection.md)
