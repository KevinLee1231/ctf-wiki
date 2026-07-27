---
type: technique
tags: [osint, account, username, social-media, identity]
skills: [ctf-osint]
raw:
  - ../raw/osint/social-media.md
  - ../raw/osint/SUCTF2026-CyberTrackWP.md
updated: 2026-07-27
---

# Cross-Platform Account and Public-Media Correlation

## 适用场景

从用户名、头像、帖子、游戏平台、公开媒体或 archive 关联多个账号和真实实体；核心是建立多证据身份链并控制同名误报。

## 识别信号

- 稀有 username、头像、bio 片段、发布时间或内容模板跨平台复现。
- 游戏/代码/社交平台互相链接，或 archive 保留已删除 profile。
- 媒体 EXIF、文件名、评论和互动网络提供额外标识。

## 最小证据

- 至少两个独立高辨识度属性支持同一实体。
- 每条账号边记录来源、时间和可能的替代解释。
- 区分平台公开信息与猜测/私人数据。

## 解法骨架

1. 规范化 username、大小写、历史别名和头像 hash。
2. 从公开 profile、帖子、仓库、游戏平台和 archive 扩展候选。
3. 按时间、地理、语言、关联账号和独特内容交叉验证。
4. 对同名候选保留反证，输出最短可审计身份链。

## 关键变体

- Username reuse。
- Avatar/media reverse correlation。
- Archived/deleted profile 与社交图关联。

## 常见陷阱

- 相同昵称即认定同一人。
- 只引用聚合网站，不回查平台原始页面。
- 忽略账号转手、复制头像和时间冲突。

## 关联技巧

- [osint-account-public-media-correlation.md](osint-account-public-media-correlation.md)
- [public-record-dns-whois-and-archive-pivoting.md](public-record-dns-whois-and-archive-pivoting.md)
- [visual-geolocation-and-media-metadata-correlation.md](visual-geolocation-and-media-metadata-correlation.md)

## 原始资料

- [social-media.md](../raw/osint/social-media.md)
- [SUCTF2026-CyberTrackWP](../raw/osint/SUCTF2026-CyberTrackWP.md)
