---
type: technique
tags: [osint, technique]
skills: [ctf-osint]
raw:
  - ../raw/osint/geolocation-and-media.md
updated: 2026-06-04
---

# Geolocation and Media Analysis

## 适用场景

需要从公开来源、图像、地理、账号、域名、社媒或历史快照中定位信息。

本页不是 raw 的目录页；它把原始资料中的案例压缩成可迁移的判断信号、最小证据和解题骨架。

## 识别信号

- 题目给出图片、用户名、坐标、域名、时间、社媒线索或公开页面。
- 目标是地点、身份、公开记录、历史版本或关联账号。
- 公开证据链比技术利用更重要。
- 题面或 raw 线索能落到这些关键词之一：Image Analysis、Reverse Image Search、Geolocation Techniques、MGRS (Military Grid Reference System)、Google Plus Codes / Open Location Codes (MidnightCTF 2026)、Metadata Extraction、Hardware/Product Identification、Newspaper Archives and Historical Research。

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
| Image Analysis | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Reverse Image Search | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Geolocation Techniques | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| MGRS (Military Grid Reference System) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Google Plus Codes / Open Location Codes (MidnightCTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Metadata Extraction | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Hardware/Product Identification | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Newspaper Archives and Historical Research | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Google Street View Panorama Matching (EHAX 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Road Sign Language and Driving Side Analysis (EHAX 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Post-Soviet Architecture and Brand Identification (EHAX 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| IP Geolocation and Attribution | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Google Lens Cropped Region Search (UTCTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Reflected and Mirrored Text Reading (UTCTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| What3Words (W3W) Geolocation (UTCTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Monumental Letters / Letreiro Identification (UTCTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Google Maps Crowd-Sourced Photo Verification (MidnightCTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Overpass Turbo Spatial Queries (LAB'OSINT 2025) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Music-Themed Landmark Geolocation with Key Encoding (BSidesSF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| 地理定位速查技巧族：图片、坐标、IP 与隐写线索 | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Image Analysis & Reverse Image Search | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Geolocation | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| MGRS Coordinates | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Google Plus Codes | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |

## 常见陷阱

- 只按关键词跳页，没有先构造最小证据。
- 照搬 raw 中的一次性 payload，没有检查当前题的边界条件。
- 忽略相邻技巧之间的 pivot，导致在错误方向上继续投入时间。

## 关联技巧

- [osint-account-public-media-correlation.md](osint-account-public-media-correlation.md)
- [web-and-dns.md](web-and-dns.md)
- [osint-tooling.md](osint-tooling.md)

## WP 案例沉淀

本节只抽取 raw WP 中可复用的识别信号和下一跳，不替代原始题解正文。

| Raw WP | 复用信号 | 下一跳 |
|---|---|---|
| [LilacCTF2026-sky-is-ours-wp](../raw/osint/LilacCTF2026-sky-is-ours-wp.md) | 图片、轨迹、公开媒体或社交线索，先保存证据图、查询 URL 和交叉验证路径。 | [geolocation-and-media.md](geolocation-and-media.md) |
| [SU_CyberTrackWP](../raw/osint/SU_CyberTrackWP.md) | 博客、GitHub、邮箱头像、Minecraft 昵称和社交平台线索串联，先保存每个公开身份节点和哈希/邮箱证据。 | [osint-account-public-media-correlation.md](osint-account-public-media-correlation.md) |

## 原始资料
- [geolocation-and-media.md](../raw/osint/geolocation-and-media.md)
