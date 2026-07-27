---
type: family
tags: [forensics, family, video, document, media, stego]
skills: [ctf-stego, ctf-forensics, ctf-reverse]
raw:
  - ../raw/stego/video-document-and-media-stego.md
  - ../raw/reverse/WMCTF2025-videoplayer-wp.md
updated: 2026-07-27
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
| [MoeCTF2024-我的图层在你之上-wp](../raw/stego/MoeCTF2024-我的图层在你之上-wp.md) | PDF 隐写不能只看渲染结果。应同时检查文件尾、对象流、注释与图层结构；矢量 PDF 导入编辑器后出现多个独立对象，就是继续拆层的强信号。图层合成时还要区分普通透明度叠加与逐像素 Add，本题需要后者才能还原密码图案。 |
| [UMDCTF2018-shrek-this-out-wp](../raw/stego/UMDCTF2018-shrek-this-out-wp.md) | 视频二维码题要同时保证帧完整、顺序正确和分层解码正确。先用首帧元数据核对帧数，再逐帧 Base64 解码、拼接压缩数据、解压并检查最终文件元数据，可以让每一层都有明确的结构校验。 |
| [UMDCTF2023-straight-outta-the-scif-wp](../raw/stego/UMDCTF2023-straight-outta-the-scif-wp.md) | 高分辨率保留并解析 PDF 每页的黄色打印机追踪点，再把序列号字段按两组三位十进制 ASCII 重组；打印/扫描语境下，整页规律重复的微小黄点应优先联想到 Machine Identification Code，而不是普通水印或 LSB。 |
| [UMDCTF2024-i-love-shapes-and-colors-wp](../raw/stego/UMDCTF2024-i-love-shapes-and-colors-wp.md) | 识别特定软件生成的视频数据载体，使用同一工具和已知口令逆向恢复文件；题面同时出现 AES-256、几何形状视频、已知密码和专用工具链接时，重点是格式解码而非密码破解。 |
| [WMCTF2022-nano-tv-wp](../raw/stego/WMCTF2022-nano-tv-wp.md) | 本题核心是利用 tar PaxHeader 中的高精度创建时间恢复视频帧顺序。单帧雪花图像信息量不足，按文件名播放也读不到内容，只有按真实创建时间排序后生成动图才能看到漂移文字；识别信号是附件为 tar 包、文件名顺序混乱、题面反复暗示电视雪花和“等等”、每张 PNG 看似随机但尺寸一致。 |
| [WMCTF2023-ez-v1deo-wp](../raw/stego/WMCTF2023-ez-v1deo-wp.md) | 逐帧提取视频像素 LSB 并重建可视化视频；题目给出视频且画面无明显异常时，应检查各通道最低位、帧间差分和 alpha/颜色通道。 |

## Technique 下一跳

| 首轮判断 | 具体 technique |
|---|---|
| 视频帧差、颜色通道、时间采样或轨迹隐藏数据 | [media-channel-bitplane-and-frame-difference-extraction.md](media-channel-bitplane-and-frame-difference-extraction.md) |
| 文档 revision、隐藏对象或编辑历史保留内容 | [structured-document-history-and-hidden-object-recovery.md](structured-document-history-and-hidden-object-recovery.md) |
| 媒体/文档容器存在尾随、伪装或内嵌载荷 | [file-format-and-embedded-payload-identification.md](file-format-and-embedded-payload-identification.md) |

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
