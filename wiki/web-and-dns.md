---
type: technique
tags: [osint, technique]
skills: [ctf-osint]
raw:
  - ../raw/osint/web-and-dns.md
updated: 2026-05-21
---

# Web and DNS OSINT

## 适用场景

需要从公开来源、图像、地理、账号、域名、社媒或历史快照中定位信息。

本页不是 raw 的目录页；它把原始资料中的案例压缩成可迁移的判断信号、最小证据和解题骨架。

## 识别信号

- 题目给出图片、用户名、坐标、域名、时间、社媒线索或公开页面。
- 目标是地点、身份、公开记录、历史版本或关联账号。
- 公开证据链比技术利用更重要。
- 题面或 raw 线索能落到这些关键词之一：Google Dorking、Google Docs/Sheets in OSINT、DNS Reconnaissance、DNS TXT Record OSINT、Tor Relay Lookups、GitHub Repository Comments、Telegram Bot Investigation、FEC Political Donation Research。

## 最小证据

- 已完成主方向判断，并确认本页技巧比相邻技巧更能解释当前证据。
- 至少有一个可复现输入、输出、文件结构、数学关系、协议行为或运行时状态。
- 能指出 raw 案例中哪一个变体与当前题最接近，以及不同点在哪里。

## 解法骨架

1. 提取元数据、文本、图像和上下文。
2. 按域名/账号/地理/时间线分支收集证据。
3. 多来源交叉验证。
4. 记录 URL、截图、时间戳和推理链。

## 关键变体

| 变体 | 复用重点 |
|---|---|
| Google Dorking | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Google Docs/Sheets in OSINT | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| DNS Reconnaissance | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| DNS TXT Record OSINT | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Tor Relay Lookups | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| GitHub Repository Comments | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Telegram Bot Investigation | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| FEC Political Donation Research | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Wayback Machine | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| WHOIS Investigation | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Shodan SSH Fingerprint Lookup (EKOPARTY CTF 2016) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Shodan SSH Fingerprint Lookup | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Fake Service Banner Detection via Fingerprinting (MetaCTF Flash 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Fake Service Banner Detection | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Git Commit Author Mining for Credentials (Hackover 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| .DSStore Directory Enumeration with Python-dsstore (35C3 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| TTF Glyph Contour Diffing for Obfuscated CAPTCHA (Square CTF 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Cross-Challenge Container IP Reuse (RITSEC 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Web / DNS OSINT 速查技巧族：公开索引、历史快照与仓库痕迹 | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Resources | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| String Identification | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Google Docs/Sheets | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| GitHub Repository Analysis | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |

## 常见陷阱

- 只按关键词跳页，没有先构造最小证据。
- 照搬 raw 中的一次性 payload，没有检查当前题的边界条件。
- 忽略相邻技巧之间的 pivot，导致在错误方向上继续投入时间。

## 关联技巧

- [geolocation-and-media.md](geolocation-and-media.md)
- [osint-account-public-media-correlation.md](osint-account-public-media-correlation.md)
- [osint-tooling.md](osint-tooling.md)

## 原始资料

- [web-and-dns.md](../raw/osint/web-and-dns.md)
