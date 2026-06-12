---
type: technique
tags: [web, technique]
skills: [ctf-web]
raw:
  - ../raw/web/polyglot-url-tricks-and-ssrf-leaks.md
updated: 2026-05-21
---

# Polyglot, URL Tricks and SSRF Leaks

## 适用场景

HTTP 行为、认证授权、服务端解析差异、文件/路径处理、模板或浏览器执行模型是主要障碍。

本页不是 raw 的目录页；它把原始资料中的案例压缩成可迁移的判断信号、最小证据和解题骨架。

## 识别信号

- 请求、路由、Cookie、Header、body 或服务端状态出现可控差异。
- 源码或响应能定位到中间件、鉴权、解析器、模板、上传或内部 API。
- 能用最小 HTTP payload 复现行为变化。
- 题面或 raw 线索能落到这些关键词之一：WAV Polyglot Upload Bypass via .wave Extension (PlaidCTF 2018)、Multi-Slash URL Parser path.startswith Bypass (CSAW 2018 Finals)、Xalan XSLT math:random() Seed Guess (35C3 2018)、SoapClient useragent CRLF Method Smuggling (35C3 2018)、gopher:// No-Host URL Scheme Bypass (35C3 2018)、SSRF Credential Leak via Attacker-Specified Outbound URL (ASIS Finals 2018)。

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
| WAV Polyglot Upload Bypass via .wave Extension (PlaidCTF 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Multi-Slash URL Parser path.startswith Bypass (CSAW 2018 Finals) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Xalan XSLT math:random() Seed Guess (35C3 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| SoapClient useragent CRLF Method Smuggling (35C3 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| gopher:// No-Host URL Scheme Bypass (35C3 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| SSRF Credential Leak via Attacker-Specified Outbound URL (ASIS Finals 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |

## 常见陷阱

- 只按关键词跳页，没有先构造最小证据。
- 照搬 raw 中的一次性 payload，没有检查当前题的边界条件。
- 忽略相邻技巧之间的 pivot，导致在错误方向上继续投入时间。

## 关联技巧

- [artifact-trust-ssrf-to-node-require-rce.md](artifact-trust-ssrf-to-node-require-rce.md)
- [auth-bypass-cookies-and-hidden-routes.md](auth-bypass-cookies-and-hidden-routes.md)
- [auth-edge-cases-and-protocol-bypasses.md](auth-edge-cases-and-protocol-bypasses.md)
- [auth-jwt.md](auth-jwt.md)
- [csp-xsleak-and-browser-exfiltration.md](csp-xsleak-and-browser-exfiltration.md)

## 原始资料

- [polyglot-url-tricks-and-ssrf-leaks.md](../raw/web/polyglot-url-tricks-and-ssrf-leaks.md)
