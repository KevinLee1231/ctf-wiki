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
updated: 2026-07-27
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

## Technique 下一跳

| 首轮判断 | 具体 technique |
|---|---|
| username/avatar/post/game profile 需跨平台建立身份链 | [cross-platform-account-and-public-media-correlation.md](cross-platform-account-and-public-media-correlation.md) |
| DNS/WHOIS/CT/Archive/repo/document metadata 提供 pivot | [public-record-dns-whois-and-archive-pivoting.md](public-record-dns-whois-and-archive-pivoting.md) |
| 图片/视频/坐标/地标与账号媒体需联合定位 | [visual-geolocation-and-media-metadata-correlation.md](visual-geolocation-and-media-metadata-correlation.md) |

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
| [0xGame2025-week3-我是NaeS姐姐的舔狗-wp](../raw/osint/0xGame2025-week3-我是NaeS姐姐的舔狗-wp.md) | 本题考查跨平台身份关联与证据闭环。有效线索不是孤立的：博客的居住地与站台照片共同确认车站；Grav 重置提示中的 QQ 邮箱连接到 QQ 动态；昵称和生日把 QQ 转发视频中的评论者关联到 B 站账号；登机牌则优先依赖 PDF417 的结构化数据，而不是只凭模糊的时刻猜航班。 |
| [ACTF2025-master-of-movie-wp](../raw/osint/ACTF2025-master-of-movie-wp.md) | 电影截图 OSINT 的核心是选择高区分度线索。人物脸部和普通场景适合反向图片搜索；反向搜索失败时，应转向画面中的稀有文本，并从“语言—场景类型—专名—角色或剧情”逐层验证。Hard_2 依靠韩文展签恢复学校名，Hard_0 则利用成绩榜上的罕见姓氏组合命中角色表。外部工具只负责生成候选，最终结论仍需由截图细节和权威作品信息共同支撑。 |
| [MoeCTF2024-时光穿梭机-wp](../raw/osint/MoeCTF2024-时光穿梭机-wp.md) | 历史 OSINT 的常见难点是名称漂移：同一人名可能出现旧式罗马化、现代拼音和中文名。应先用日期、报刊名、版面标题等稳定字段锁定史料，再做人物名映射；得到墓址后才进入地图核验。正文保留了史料标题与日期，因此即使原检索页面以后失效，仍能理解该链接提供的关键证据。 |
| [UMDCTF2022-ketchup-wp](../raw/osint/UMDCTF2022-ketchup-wp.md) | 人脸 OSINT 与普通相似图片搜索解决的问题不同。第一阶段用人脸特征跨图片找到同一人物，第二阶段用来源更明确的照片匹配包含姓名的网页。遇到搜索结果只显示缩略图时，页面源代码和网络请求常能恢复原图 URL；但最终结论仍应由出现姓名的独立页面交叉确认，而不是只凭相似度排名。 |
| [UMDCTF2023-tcc-2-wp](../raw/osint/UMDCTF2023-tcc-2-wp.md) | 把站点枚举、公开简历、赛事记录和社交平台消息组合成多源验证；站点地图、异常命名的 XML、公开文档和响应头都可能是题目刻意留下的导航或反馈机制。 |
| [UMDCTF2026-hades-group-wp](../raw/osint/UMDCTF2026-hades-group-wp.md) | 这道题的重点是从匿名频道内容寻找可归属的外部对象，而不是对所有群成员做无差别查询。贴纸包名连接到创建者身份，用户名历史、两套泄漏库和缓存资料用于交叉验证，区域文档库再把号码转换为实名，最终才查询记录号。题目明确限定所有身份均为虚构种子，实际 OSINT 中也应遵守授权范围并避免查询真实个人。 |
| [WMCTF2023-find-me-wp](../raw/osint/WMCTF2023-find-me-wp.md) | 把 OSINT 线索链和流量加密还原结合起来，利用密码复用解锁博客文章，再恢复通信加密；题面给出用户名和社交平台时，应检查同名 GitHub、博客、头像来源、自动化脚本和复用凭据。 |

## 原始资料

- [social-media.md](../raw/osint/social-media.md)
- [SUCTF2026-CyberTrackWP](../raw/osint/SUCTF2026-CyberTrackWP.md)
- [RCTF2025-speak-softly-love-wp](../raw/osint/RCTF2025-speak-softly-love-wp.md)
- [RCTF2025-wanna-feel-love-wp](../raw/osint/RCTF2025-wanna-feel-love-wp.md)
- [SUCTF2026-SigninWP](../raw/osint/SUCTF2026-SigninWP.md)
