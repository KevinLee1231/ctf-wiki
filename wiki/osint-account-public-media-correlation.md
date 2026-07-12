---
type: family
tags: [osint, family, account-correlation, public-media]
skills: [ctf-osint]
raw:
  - ../raw/osint/social-media.md
  - ../raw/osint/SUCTF2026-CyberTrackWP.md
  - ../raw/osint/RCTF2025-speak-softly-love-wp.md
  - ../raw/osint/RCTF2025-wanna-feel-love-wp.md
  - ../raw/osint/SUCTF2026-SigninWP.md
updated: 2026-07-06
---

# OSINT Account and Public Media Correlation

## 作用边界

题目要求从公开账号、主页、历史记录、公开媒体、游戏平台、邮箱头像、代码提交、社交平台或时间线中定位身份、地点、flag-like 字符串或下一跳线索。本页是 OSINT 身份链 family，核心不是“社媒搜索”本身，而是把多个公开身份节点连成可验证证据链。

## 识别信号

- 题面给出用户名、邮箱、头像、主页、GitHub commit、Discord/Minecraft/CTFtime、公开视频、音乐、gopher、Wayback/Ghostarchive 或平台昵称。
- flag 不一定藏在单一页面，常常需要跨平台拼出同一人、同一时间线、同一作品或同一站点迁移。
- 线索包含可稳定复查的公开证据：URL、commit hash、用户名历史、头像 hash、视频发布时间、站点快照、支付/墓碑/团队页记录。
- 同时出现媒体隐写和 OSINT 时，先判断媒体是承载 flag，还是只是把你引向公开身份链。

## 最小证据

- 保存每个公开来源的 URL、访问时间和关键字段，避免只记录结论。
- 至少有两个独立节点能互相支持同一身份或同一线索，例如邮箱头像对应 GitHub、博客链接到 Discord、视频描述指向个人主页。
- 对会变化的平台内容，优先查快照、镜像、archive、commit history 或公开 API。
- 能说明当前结果是 flag、下一跳线索，还是需要回到媒体/文件取证继续处理。

## 分流流程

1. 把题面所有可见实体拆成账号、域名、邮箱、媒体、时间和文本片段。
2. 先查稳定来源：Git commit、站点快照、公开 API、团队页、历史昵称，再查易变化社媒页面。
3. 用相同头像、用户名变体、邮箱 hash、个人主页互链、时间线和平台 ID 建立身份链。
4. 对每个节点记录证据来源；遇到媒体文件时交给相邻的地理/媒体/隐写页面继续验证。
5. 最后只提交能被复查的 flag-like 字符串或明确下一跳，不把猜测当结论。

## 账号证据路线分流

| 公开证据形态 | 关联判断 |
|---|---|
| 多平台账号关联 | 用户名、头像、邮箱、个人主页和平台 ID 相互佐证；不要只凭一个昵称命中就下结论。 |
| GitHub / commit / 博客链 | commit、issue、README、历史页面和博客外链常保留删除前线索，先查历史再查当前页面。 |
| 游戏平台和社区账号 | Minecraft、Discord、CTFtime、Steam 等平台常用历史昵称、团队页或公开 API 作为稳定证据。 |
| 公开视频和公开媒体 | 视频描述、音频文件名、发布账号、字幕、评论和外链可以比媒体内容本身更关键。 |
| Archive / mirror / gopher | 页面失效时优先保存 archive、Ghostarchive、gopher 或站点镜像，再继续串联。 |
| 媒体隐写混合 | 如果图片/音频/视频同时出现异常 artifact，先分离“公开来源证据”和“文件内容恢复”两条线。 |

## 合并与拆分结论

- 保留为 `family`：raw 覆盖社交账号、游戏平台、公开媒体、GitHub/博客、archive、Unicode 隐写和平台 API，重点是身份节点之间的 pivot。
- 不并入 [geolocation-and-media.md](geolocation-and-media.md)：地理页验证地点，本页验证公开身份链和账号关系。
- 不拆 Twitter/BlueSky/Discord/Gaming 等小页：目前这些平台线索经常共同服务于同一身份链，拆开会破坏证据闭合。

## 常见陷阱

- 只截图搜索结果，不保存可复查 URL、时间和字段。
- 把同名账号当同一人，没有交叉验证头像、主页、commit 或时间线。
- 在公开页面找不到结果后直接放弃，没有查历史快照、commit history 或平台 API。
- 把所有媒体题都当隐写处理，忽略媒体的发布者、描述、评论和外链。

## 关联技巧

- [geolocation-and-media.md](geolocation-and-media.md)
- [web-and-dns.md](web-and-dns.md)
- [video-document-and-media-stego.md](video-document-and-media-stego.md)
- [pdf-png-gif-and-text-stego.md](pdf-png-gif-and-text-stego.md)
- [osint-tooling.md](osint-tooling.md)

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [SUCTF2026-CyberTrackWP](../raw/osint/SUCTF2026-CyberTrackWP.md) | 博客、GitHub commit、邮箱头像、Minecraft 历史昵称、社交平台和 Discord 需要按身份节点闭合证据链。 |
| [RCTF2025-speak-softly-love-wp](../raw/osint/RCTF2025-speak-softly-love-wp.md) | 音乐视频、个人主页、音频文件、SVN 和 gopher 页面串联时，先保存每个公开 URL 和时间线。 |
| [RCTF2025-wanna-feel-love-wp](../raw/osint/RCTF2025-wanna-feel-love-wp.md) | 多阶段 OSINT 可同时包含隐写、媒体元数据、购买记录和墓碑信息，避免只按音频题处理。 |
| [SUCTF2026-SigninWP](../raw/osint/SUCTF2026-SigninWP.md) | 签到题给公开团队页面时，应先检查 CTFtime/主页等公开资料里的 flag-like 字符串。 |

## 原始资料

- [social-media.md](../raw/osint/social-media.md)
- [SUCTF2026-CyberTrackWP](../raw/osint/SUCTF2026-CyberTrackWP.md)
- [RCTF2025-speak-softly-love-wp](../raw/osint/RCTF2025-speak-softly-love-wp.md)
- [RCTF2025-wanna-feel-love-wp](../raw/osint/RCTF2025-wanna-feel-love-wp.md)
- [SUCTF2026-SigninWP](../raw/osint/SUCTF2026-SigninWP.md)
