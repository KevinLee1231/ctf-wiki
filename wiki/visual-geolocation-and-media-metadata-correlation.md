---
type: technique
tags: [osint, geolocation, media, metadata, landmarks]
skills: [ctf-osint]
raw:
  - ../raw/osint/geolocation-and-media.md
  - ../raw/osint/LilacCTF2026-sky-is-ours-wp.md
updated: 2026-07-27
---

# Visual Geolocation and Media-Metadata Correlation

## 适用场景

从图片/视频中的路牌、地标、道路、阴影、天气、航线或 EXIF/坐标/时间信息定位地点，并用地图、街景和独立公开来源验证。

## 识别信号

- 可见文字、道路标线、建筑、山形、交通规则或独特设施。
- EXIF、文件名、时间、GPS、航班/卫星/天气等 metadata 可用。
- 多帧视频提供行进方向、相邻地标或时间变化。

## 最小证据

- 分离“图中直接观察”与“外部来源推断”。
- 候选地点至少匹配两个独立视觉/metadata 特征。
- 保存地图/街景坐标、方向、时间和对照画面。

## 解法骨架

1. 提取原始 metadata 和清晰帧，OCR 所有可见文字。
2. 按语言、道路、地貌、建筑和太阳/天气逐步缩小区域。
3. 用地图、街景、航拍和公开媒体逐项对齐。
4. 记录排除候选的反证，最后给出可复核坐标/地点。

## 关键变体

- Landmark/road-sign geolocation。
- EXIF/GPS/time/weather correlation。
- Video route/frame sequence localization。

## 常见陷阱

- 过度依赖单一相似地标。
- 使用被平台重写的 metadata 当原始证据。
- 只给地点猜测，没有视角/方向对照。

## 关联技巧

- [geolocation-and-media.md](geolocation-and-media.md)
- [public-record-dns-whois-and-archive-pivoting.md](public-record-dns-whois-and-archive-pivoting.md)
- [cross-platform-account-and-public-media-correlation.md](cross-platform-account-and-public-media-correlation.md)

## 原始资料

- [geolocation-and-media.md](../raw/osint/geolocation-and-media.md)
- [LilacCTF2026-sky-is-ours-wp](../raw/osint/LilacCTF2026-sky-is-ours-wp.md)
