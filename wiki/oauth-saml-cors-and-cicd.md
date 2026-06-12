---
type: technique
tags: [web, technique]
skills: [ctf-web]
raw:
  - ../raw/web/oauth-saml-cors-and-cicd.md
updated: 2026-05-21
---

# OAuth, SAML, CORS and CI/CD

## 适用场景

HTTP 行为、认证授权、服务端解析差异、文件/路径处理、模板或浏览器执行模型是主要障碍。

本页不是 raw 的目录页；它把原始资料中的案例压缩成可迁移的判断信号、最小证据和解题骨架。

## 识别信号

- 请求、路由、Cookie、Header、body 或服务端状态出现可控差异。
- 源码或响应能定位到中间件、鉴权、解析器、模板、上传或内部 API。
- 能用最小 HTTP payload 复现行为变化。
- 题面或 raw 线索能落到这些关键词之一：OAuth/OIDC Exploitation、Open Redirect Token Theft、OIDC ID Token Manipulation、OAuth State Parameter CSRF、CORS Misconfiguration、Git History Credential Leakage (Barrier HTB)、CI/CD Variable Credential Theft (Barrier HTB)、Identity Provider API Takeover (Barrier HTB)。

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
| OAuth/OIDC Exploitation | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Open Redirect Token Theft | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| OIDC ID Token Manipulation | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| OAuth State Parameter CSRF | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| CORS Misconfiguration | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Git History Credential Leakage (Barrier HTB) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| CI/CD Variable Credential Theft (Barrier HTB) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Identity Provider API Takeover (Barrier HTB) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| SAML SSO Flow Automation (Barrier HTB) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Apache Guacamole Connection Parameter Extraction (Barrier HTB) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Login Page Poisoning for Credential Harvesting (Watcher HTB) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| TeamCity REST API RCE (Watcher HTB) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Base64 Decode Leniency and Parameter Override for Signature Bypass (BCTF 2016) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Hash Length Extension Attack (ASIS CTF 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |

## 常见陷阱

- 只按关键词跳页，没有先构造最小证据。
- 照搬 raw 中的一次性 payload，没有检查当前题的边界条件。
- 忽略相邻技巧之间的 pivot，导致在错误方向上继续投入时间。

## 关联技巧

- [workflow-runner-internal-api-chain.md](workflow-runner-internal-api-chain.md)
- [path-confusion-to-signed-internal-request-chain.md](path-confusion-to-signed-internal-request-chain.md)
- [artifact-trust-ssrf-to-node-require-rce.md](artifact-trust-ssrf-to-node-require-rce.md)
- [auth-bypass-cookies-and-hidden-routes.md](auth-bypass-cookies-and-hidden-routes.md)
- [auth-edge-cases-and-protocol-bypasses.md](auth-edge-cases-and-protocol-bypasses.md)

## 原始资料

- [oauth-saml-cors-and-cicd.md](../raw/web/oauth-saml-cors-and-cicd.md)
