---
type: family
tags: [forensics, family, video, document, media, stego]
skills: [ctf-stego, ctf-forensics, ctf-reverse]
raw:
  - ../raw/stego/video-document-and-media-stego.md
  - ../raw/reverse/WMCTF2025-videoplayer-wp.md
updated: 2026-06-12
---

# Video, Document and Media Stego

## 作用边界

本页是视频、文档、容器媒体和跨媒体隐写 family。它覆盖帧叠加/平均、倒放音频、JPEG XL TOC permutation、Arnold cat map、SSTV FM、MJPEG extra bytes、EXIF zlib、PDF xref covert channel、ANSI escape、像素级 ECB 去重和多色 QR 映射。

如果关键证据在运行时解密出的媒体 buffer，本页与 reverse runtime 页联动：先 dump 真实媒体，再做取证分析。

## 识别信号

- 附件是视频、PDF、Office、MJPEG、JXL、EXIF、ANSI 终端录制、动态图像或自定义媒体容器。
- 单帧/单页无明显信息，但帧间平均、叠加、差分、末帧、附加字节或索引结构异常。
- 媒体需要先由播放器或程序解密，原文件本身不是最终容器。
- 图像/视频中出现 QR、条码、像素置换、颜色映射、DCT/LSB/metadata 或 xref 异常。

## 最小证据

- 容器格式、帧数、分辨率、时间轴、关键帧、附加数据位置和 metadata。
- 对变换类题，确认置换/映射是否可逆以及迭代次数或搜索边界。
- 对 PDF/文档，先检查 xref、object stream、embedded file、annotation、font、script 和外部资源。
- 对运行时解密媒体，先获得真实媒体文件并验证 magic/播放器可读性。

## 路由表

| 证据 | 先验证 | 下一跳 |
|---|---|---|
| 帧叠加/平均 | 帧对齐、透明度、运动区域和背景稳定性 | 导出帧后平均/差分 |
| MJPEG/JPEG extra bytes | `FFD9` 后是否有附加数据或多图拼接 | carving 或按帧提取尾部 |
| JXL/PNG/JPEG 元结构 | TOC、chunk、EXIF zlib、非默认 LSB 模式 | 转图像隐写 family |
| PDF xref covert | xref offset、object 顺序和增量更新是否异常 | 重建对象序列 |
| ANSI escape capture | 控制码是否构成隐藏画面或文本 | 终端渲染/清洗两路对比 |
| 多色 QR | 颜色到 bit 的映射未知 | 枚举映射并校验 QR 格式 |
| 加密自定义媒体 | 播放器运行时返回真实 buffer | 转 [runtime-patching-oracles-and-tracing.md](runtime-patching-oracles-and-tracing.md) |

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [WMCTF2025-videoplayer-wp](../raw/reverse/WMCTF2025-videoplayer-wp.md) | `.mp0` 文件由播放器使用机器信息 MD5 解密；在解密函数返回的 `std::vector` 中用头尾指针算大小并 dump，得到真实 mp4 后从视频末尾读 flag。 |

## 合并与拆分结论

- 保留为 family：视频、文档和容器媒体共享“先找容器层/帧层/对象层”的 pivot。
- 不合并进 audio 页：音频页聚焦频域和声道，本文聚焦帧、文档对象和媒体容器。
- 不合并进 image family：本文保留跨帧、文档和运行时媒体 buffer 的路线。

## 常见误判

- 只看首帧/当前页，漏掉末帧、附加字节和增量更新。
- 对视频重编码后再分析，破坏原始隐藏数据。
- PDF 只提取文字，忽略 xref、object stream 和 embedded data。
- 自定义媒体文件没先从播放器内存 dump 真实容器。

## 关联页面

- [image-bitplane-qr-and-jpeg-stego.md](image-bitplane-qr-and-jpeg-stego.md)
- [audio-frequency-and-archive-stego.md](audio-frequency-and-archive-stego.md)
- [runtime-patching-oracles-and-tracing.md](runtime-patching-oracles-and-tracing.md)
- [pdf-png-gif-and-text-stego.md](pdf-png-gif-and-text-stego.md)
- [forensics-tooling.md](forensics-tooling.md)

## 原始资料

- [video-document-and-media-stego.md](../raw/stego/video-document-and-media-stego.md)
- [WMCTF2025-videoplayer-wp](../raw/reverse/WMCTF2025-videoplayer-wp.md)
