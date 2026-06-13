---
type: family
tags: [osint, family, geolocation, media, imagery]
skills: [ctf-osint]
raw:
  - ../raw/osint/geolocation-and-media.md
updated: 2026-06-12
---

# Geolocation and Media Analysis

## 作用边界

本页是地理定位与媒体线索 family，用于从图片、视频帧、地图坐标、街景、路牌、建筑、产品、元数据、IP 地理、Plus Codes、What3Words、MGRS 和公开媒体中定位地点或下一跳线索。

如果核心是公开账号/社媒身份链，转 [osint-account-public-media-correlation.md](osint-account-public-media-correlation.md)；如果核心是域名、DNS、历史网页或公开索引，转 [web-and-dns.md](web-and-dns.md)；如果媒体文件本身存在隐写，转 forensics 媒体页面。

## 共同识别信号

- 题目给出照片、视频、街景截图、坐标、地标、路牌、语言、建筑风格、商品/硬件外观、地图片段、EXIF 或 IP。
- 目标是地点、城市、具体坐标、拍摄方向、时间、设备/产品型号或从地点得到的编码 key。
- 证据需要多来源交叉：图像内容、地图、街景、公开照片、坐标系统、天气/时间、语言和道路规则。

## 最小证据

- 保存原始媒体、裁剪区域、搜索关键词、地图 URL、候选坐标和查询时间。
- 对地点结论至少有两个独立证据：路牌/文字、建筑/地标、街景匹配、EXIF、坐标编码、公开照片或周边 POI。
- 对坐标编码，确认格式和精度：lat/lon、MGRS、Plus Code、What3Words、IP geo 都有不同误差边界。
- 对图像搜索，记录是全图命中、局部 crop 命中、镜像/反射文本还是同一地点的相似图片。

## 首轮路由

| 证据形态 | 先做什么 | 下一跳 |
|---|---|---|
| 照片/视频帧含地标、路牌、建筑 | 先裁剪关键区域做反搜，再用街景、道路方向、语言和建筑风格交叉验证。 | [osint-tooling.md](osint-tooling.md) |
| EXIF/metadata/IP | 提取坐标、设备、时间、软件和网络归属；IP geo 只能作为弱线索，需要二次验证。 | [web-and-dns.md](web-and-dns.md) |
| MGRS/Plus Code/What3Words | 先确认编码系统和精度，再落到地图并验证周边对象。 | [osint-tooling.md](osint-tooling.md) |
| 镜像/反射/模糊文字 | 先翻转、增强、局部 OCR 和关键词 wildcard，再进入图像搜索。 | [pdf-png-gif-and-text-stego.md](pdf-png-gif-and-text-stego.md) |
| 地点推导出 key/字符 | 先确认地点，再把地标、音乐、键号、邮编或坐标映射为编码输入。 | [encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md) |
| 发布账号/视频描述更关键 | 保留媒体来源页面和账号链，转账号相关性页面继续。 | [osint-account-public-media-correlation.md](osint-account-public-media-correlation.md) |

## 合并与拆分结论

- 保留为 `family`：raw 覆盖图像反搜、地理坐标、街景、路牌/建筑、IP、产品识别、历史媒体和编码 key，路线分叉明显。
- 不并入账号相关性页：本页核心是地点/媒体内容验证，账号页核心是公开身份链闭合。
- 不拆 MGRS/Plus Code/Street View 等小页：当前 raw 多为同一地理定位流程中的变体，拆分会让 OSINT 入口碎片化。

## 常见陷阱

- 只凭一个相似图片下结论，没有用街景、文字、道路方向或周边 POI 验证。
- 把 IP geolocation 当精确坐标；它通常只能缩小区域。
- 反射文字没先镜像，或只搜全图不搜局部 crop。
- 忽略公开媒体的发布账号、描述、评论和时间线。
- 地点推导出的编码 key 没有记录映射规则，导致结果不可复查。

## 关联技巧

- [osint-account-public-media-correlation.md](osint-account-public-media-correlation.md)
- [web-and-dns.md](web-and-dns.md)
- [pdf-png-gif-and-text-stego.md](pdf-png-gif-and-text-stego.md)
- [video-document-and-media-stego.md](video-document-and-media-stego.md)
- [encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md)
- [osint-tooling.md](osint-tooling.md)

## WP 案例沉淀

| Raw WP | 复用信号 | 下一跳 |
|---|---|---|
| [LilacCTF2026-sky-is-ours-wp](../raw/osint/LilacCTF2026-sky-is-ours-wp.md) | 图片、轨迹、公开媒体或社交线索，先保存证据图、查询 URL 和交叉验证路径。 | [geolocation-and-media.md](geolocation-and-media.md) |
| [SU_CyberTrackWP](../raw/osint/SU_CyberTrackWP.md) | 博客、GitHub、邮箱头像、Minecraft 昵称和社交平台线索串联，先保存每个公开身份节点和哈希/邮箱证据。 | [osint-account-public-media-correlation.md](osint-account-public-media-correlation.md) |

## 原始资料

- [geolocation-and-media.md](../raw/osint/geolocation-and-media.md)
