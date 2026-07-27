---
type: family
tags: [stego, family, image, qr, jpeg, bitplane]
skills: [ctf-stego, ctf-forensics]
raw:
  - ../raw/stego/image-bitplane-qr-and-jpeg-stego.md
  - ../raw/stego/HGAME2026-shiori不想找女友-wp.md
  - ../raw/stego/HGAME2026-invest-on-matrix-wp.md
  - ../raw/stego/D3CTF2025-d3rpg-signin-wp.md
  - ../raw/crypto/VNCTF2026-mymnemonic-wp.md
updated: 2026-07-27
---

# Image Bitplane, QR and JPEG Stego

## 作用边界

本页是图像隐写和图像重组 family，覆盖 JPEG 量化表/DCT/slack/thumbnail、BMP/PNG bitplane、调色板、QR tile、像素置换、拼图重组、GIF/AVI 帧和 steghide 口令线索等路线。

它不替代 [pdf-png-gif-and-text-stego.md](pdf-png-gif-and-text-stego.md) 的跨格式入口；本页负责进入图像后，判断该从像素、压缩结构、二维码结构、帧序列还是元数据口令继续。

## 识别信号

- 附件是 JPEG/PNG/BMP/GIF/AVI 或由图片碎片、二维码块、缩略图、调色板、DCT 系数、低位平面组成。
- 正常预览没有 flag，但元数据、通道、bitplane、DQT/PLTE、帧差、缩放残留或坐标链存在异常。
- 需要导出中间图像、重组 QR、提取 bitstream、修复 header/chunk 或再喂给 steghide/zbar/OCR。

## 最小证据

- 保留原始文件 hash 和未重压缩副本。
- 确认格式层：文件头、chunk/marker、EXIF、调色板、帧数、缩略图、尺寸和颜色模式。
- 对 bitplane/像素类，记录通道、bit 位、扫描顺序、阈值和端序。
- 对 QR/拼图类，记录网格尺寸、finder pattern、tile 顺序、旋转和 ECC/decoder 结果。

## 首轮路由

| 证据形态 | 首轮判断 | 下一跳 |
|---|---|---|
| JPEG DQT、DCT 系数、F5、slack、thumbnail | 先解析 marker 和未引用数据，再决定 bitstream、OCR 或 steghide | [forensics-tooling.md](forensics-tooling.md) |
| BMP/PNG LSB、bitplane、RGB parity、palette unused entry | 先按通道和 bit 位导出平面，检查是否形成文本、QR 或压缩包 | [encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md) |
| QR tile、nested resize、multi-bitplane QR | 先恢复 finder/timing/网格，再用 zbar 或 known prefix 校验 | [exotic-encodings-and-file-formats.md](exotic-encodings-and-file-formats.md) |
| 图像拼图、edge matching、坐标链 | 先建立边缘相似度或坐标遍历顺序，再导出重组图 | [interactive-containers-jails-and-solvers.md](interactive-containers-jails-and-solvers.md) |
| GIF/AVI 帧差、PLTE 拼接、视频帧藏文件 | 先导出帧和 palette/chunk，再判断是否转视频/容器页面 | [video-document-and-media-stego.md](video-document-and-media-stego.md) |
| PNG magic/chunk 损坏或大小写异常 | 先修 magic、critical chunk 和 CRC，再继续隐写分析 | [file-signatures-and-flag-artifact-hunting.md](file-signatures-and-flag-artifact-hunting.md) |

## Technique 下一跳

| 首轮判断 | 具体 technique |
|---|---|
| RGB/A、bitplane、像素顺序或帧差隐藏载荷 | [media-channel-bitplane-and-frame-difference-extraction.md](media-channel-bitplane-and-frame-difference-extraction.md) |
| QR/网格符号被分片、旋转、反色或多层重组 | [qr-and-structured-symbol-reassembly.md](qr-and-structured-symbol-reassembly.md) |
| 图像只是容器，真实 payload 位于 chunk、尾随或内嵌对象 | [file-format-and-embedded-payload-identification.md](file-format-and-embedded-payload-identification.md) |

## 合并与拆分结论

本页应保留为 family。JPEG 结构、bitplane、QR 重组、帧差和图像拼图的第一步证据不同；但它们都属于图像证据源内的二级分流。媒体通道、QR 结构和容器载荷已落到三个共享 technique，避免再按单一格式制造孤立页。

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [HGAME2026-shiori不想找女友-wp](../raw/stego/HGAME2026-shiori不想找女友-wp.md) | EXIF 中的 JSON 参数直接描述像素抽样网格；先按 `start/step/column` 重排隐藏图像，再处理恢复出的线索。 |
| [HGAME2026-invest-on-matrix-wp](../raw/stego/HGAME2026-invest-on-matrix-wp.md) | QR 信息被拆成 25 组带编号的 `5x5` 二值块并隐藏在矩阵序列中；按块坐标复原 `25x25` 网格，再恢复二维码载荷。 |
| [D3CTF2025-d3rpg-signin-wp](../raw/stego/D3CTF2025-d3rpg-signin-wp.md) | RPG 场景把线索藏在标牌、异常路径、地板明暗块和位置关系中；固定地图状态后按视觉与空间证据逐层提取，摩斯等表示层再转 Crypto 解码。 |
| [VNCTF2026-mymnemonic-wp](../raw/crypto/VNCTF2026-mymnemonic-wp.md) | 图片末尾黑白格按固定像素步长转 bitstream，图像阶段只负责稳定提取 192-bit ENT，后续 BIP39 checksum/seed 交给 crypto 长尾结构页。 |
| [D3CTF2022-badw3ter-wp](../raw/stego/D3CTF2022-badw3ter-wp.md) | 图片图层、黑底 QR 和隐写链组合，先分离图层/背景再恢复二维码或图像隐藏信息。 |
| [D3CTF2023-d3image-wp](../raw/forensics/D3CTF2023-d3image-wp.md) | 图像/二维码/摩斯或像素模式是主线，先分离颜色层、定位图案再恢复可读编码。 |
| [D3CTF2025-d3image-wp](../raw/ai-ml/D3CTF2025-d3image-wp.md) | 图像块变换和隐写编码可逆，先抽出块顺序/颜色关系再写正反向恢复脚本。 |
| [0xGame2022-week3-Time-To-Live-wp](../raw/stego/0xGame2022-week3-Time-To-Live-wp.md) | 本题串联了 TTL 低位模式、Base64 文件恢复和频域水印。提取前必须先验证固定低 4 位这一统计特征，不能只因题目名就盲取某几位；位串转字节时还要指定完整长度，避免整数转换丢失前导零。JPEG 正常可见不代表分析结束，FFT 幅度谱中的规则文字才是第二层载荷。 |
| [0xGame2022-week4-SIMPLE-QR-wp](../raw/stego/0xGame2022-week4-SIMPLE-QR-wp.md) | 本题把信息分别藏在视觉层、PNG 容器层和 QR 码字层。检查二维码附件时，不应只尝试扫码，还要核对颜色、定位块、PNG 块序列、文件尾拼接和解压后数据尺寸；遇到原始 QR 码字，则按“模式标识 → 字符计数 → 定长载荷”的顺序解析，避免把终止与纠错相关数据误当成明文。 |
| [ACTF2022-ffsk-wp](../raw/stego/ACTF2022-ffsk-wp.md) | 数据链为“双路 Bell 103 FSK → Manchester → Hamming 单错纠正 → 低位优先字符帧 → Base64 PNG → QR”。每层都应用频率峰、采样长度、合法码字、syndrome 范围和 PNG data URI 等不变量验证，不能把工具乱码当作解码结果。 |
| [ACTF2022-tang-keke-temptation-wp](../raw/stego/ACTF2022-tang-keke-temptation-wp.md) | 先定位原始尺寸封面图并确认 DQT、DHT 与尺寸一致，再直接比较量化 DCT 系数；亮度块 Zigzag 前六项的 $\pm1$ 变化按“改变为 1、未改变为 0”及 MSB-first 打包。不要先解码到 RGB/YCrCb，以免舍入误差破坏系数域载荷。 |
| [ACTF2025-QQQRcode-wp](../raw/stego/ACTF2025-QQQRcode-wp.md) | 构造的核心是“一个体素尽可能同时服务三个投影”：先取三重黑点交会，再根据三条射线的覆盖计数贪心删去冗余。二维码纠错允许最终投影存在少量误差，因此不必求解严格的三维离散层析最优问题。实现中最容易出错的是坐标含义、数组轴顺序和序列化顺序，提交前应在本地用与服务端完全相同的投影及解码代码做一次端到端验证。 |
| [SekaiCTF2026-impossible-stego-wp](../raw/stego/SekaiCTF2026-impossible-stego-wp.md) | 日志本身才是证据层：它不仅保存自然语言回答，还保存每次工具调用里的完整源码和修改参数。面对“AI 生成工具 + 最终产物”的题，应先检查代理、网关和会话日志是否重放了全量上下文，再考虑逆向像素算法。恢复时要区分被弃用的首版与最终包，并利用 magic、HMAC 和 CRC32 三层校验确认源码版本及提取结果一致。 |
| [UMDCTF2017-coffee-wp](../raw/stego/UMDCTF2017-coffee-wp.md) | 位平面载荷不一定放在整体 RGB 的最低位，也不一定从字节边界 0 开始。系统检查时应枚举通道、位号、像素遍历和 bit 打包顺序，并用 gzip、PNG、ZIP 等完整头部字段验证候选。本题由 gzip 结构和官方 SHA-256 构成了完整证据链。 |
| [UMDCTF2017-hey-you-guys-wp](../raw/stego/UMDCTF2017-hey-you-guys-wp.md) | PNG 隐写不仅可以利用像素、调色板和附加块，也可以利用“块长度”这种结构元数据。看到大量异常短、连续的同类型块时，应把长度、类型、CRC 或块顺序分别尝试为编码载体。本题无需解压 IDAT 内容。 |
| [UMDCTF2018-desktop-background-wp](../raw/stego/UMDCTF2018-desktop-background-wp.md) | 能正常打开的 PNG 仍可能在 `IEND` 后携带附加数据。发现第二个透明图层后，不应只单独查看，还要依据相同尺寸和坐标系叠加到原图；本题的红框只有在叠加后才构成完整的选字节规则。 |
| [UMDCTF2019-jogging-around-bethesda-wp](../raw/stego/UMDCTF2019-jogging-around-bethesda-wp.md) | 条码解码失败时，需要区分图像质量问题和格式版本不兼容。这里保留固定历史提交链接，因为它是复现所需的稳定源码依据；同时已在正文中说明 JAB Code 的性质、版本差异和具体提交，即使不访问外链也能理解关键机制。 |
| [UMDCTF2020-cool-coin-wp](../raw/stego/UMDCTF2020-cool-coin-wp.md) | 图像主题往往直接提示采样序列。本题决定性线索不是硬币，而是埃拉托色尼；从素数位置取位后再按字节重组，才是与题意一致的隐藏通道。 |
| [UMDCTF2020-kuler-wp](../raw/stego/UMDCTF2020-kuler-wp.md) | 把颜色常量视作整数而不是视觉配色后，规律就变成固定宽度位流。重组二维码时要同时处理每行填充和像素纵横比，缩放必须使用最近邻，避免插值产生灰边。 |
| [UMDCTF2022-blue-wp](../raw/stego/UMDCTF2022-blue-wp.md) | 这类题不应只寻找最低有效位或固定通道。已知生成逻辑时，应先找在随机扰动下仍保持不变的统计量。本题的位置和通道都随机，但每次操作贡献的总增量恒为一，所以按行求 RGB 差值总和即可消除随机性。参考像素必须选在脚本明确不会改动的位置，否则会给整行引入系统偏差。 |
| [UMDCTF2022-squarsa-wp](../raw/stego/UMDCTF2022-squarsa-wp.md) | 彩色噪声中出现规则方块时，应分别检查 RGB/Alpha 通道；多个二值载荷叠加后会掩盖各自的结构；QR 的 finder、separator、timing 和 alignment pattern 具有固定几何规范，可按版本和尺寸人工补回。 |
| [UMDCTF2025-alien-transmission-wp](../raw/stego/UMDCTF2025-alien-transmission-wp.md) | 根据固定 PRNG 种子重建卷积核，对保存信号的绿色通道撤销线性映射，再做无监督 Wiener 反卷积；图像整体仍保留轮廓，但局部呈现卷积拖影；源码中核尺寸、种子和输出通道均已固定。 |
| [WMCTF2020-dalabengba-wp](../raw/stego/WMCTF2020-dalabengba-wp.md) | RPG Maker MV 资源可用已知文件头与加密头异或恢复 key；随后把剧情提示分别映射到 Java 盲水印/Aztec、Encrypto/空白字符隐写和地图事件路径。多段 flag 应逐段记录来源、提示、工具与验证结果。 |
| [WMCTF2022-nanostego-wp](../raw/stego/WMCTF2022-nanostego-wp.md) | 本题核心是连续拆 PNG 容器和盲水印算法。先按 PNG chunk 结构找双 `IEND`，再对 `IDAT` 正常图像数据后的额外 zlib 数据解压，拿到脚本和字体，最后反写盲水印解码器。 |
| [WMCTF2023-perfect-two-way-foil-wp](../raw/stego/WMCTF2023-perfect-two-way-foil-wp.md) | 用 Hilbert 曲线在二维图片和三维体数据之间做坐标还原，再对还原切片继续做 LSB；图片呈现空间填充曲线纹理、尺寸正好是 $2^n \times 2^n$，且题面暗示“双向/三维”时，应考虑 Hilbert/Z-order 等空间重排。 |
| [WMCTF2023-steglab-pointattack-1-2-wp](../raw/stego/WMCTF2023-steglab-pointattack-1-2-wp.md) | 把隐写从“脆弱的单点 LSB”改成“带冗余或强特征的鲁棒嵌入”；题目存在 `attack.py`、PSNR 阈值、平台反复攻击图片时，应先分析攻击脚本破坏的是点、线、压缩还是格式转换。 |

## 常见陷阱

- 用截图或编辑器另存图片，破坏原始低位和 chunk。
- 只跑 steghide/binwalk，没检查 bitplane、palette、thumbnail 和 DCT。
- QR 重组只按视觉猜，不利用 finder/timing 和 ECC 反馈。
- JPEG 题只看像素，忽略 marker、DQT 和压缩域。
- 解出 bitstream 后没继续按压缩包、Base64 或 XOR 检查。

## 关联技巧

- [cross-domain-forensics-technique-map.md](cross-domain-forensics-technique-map.md)
- [pdf-png-gif-and-text-stego.md](pdf-png-gif-and-text-stego.md)
- [video-document-and-media-stego.md](video-document-and-media-stego.md)
- [exotic-encodings-and-file-formats.md](exotic-encodings-and-file-formats.md)
- [forensics-tooling.md](forensics-tooling.md)

## 原始资料

- [image-bitplane-qr-and-jpeg-stego.md](../raw/stego/image-bitplane-qr-and-jpeg-stego.md)
- [HGAME2026-shiori不想找女友-wp](../raw/stego/HGAME2026-shiori不想找女友-wp.md)
- [HGAME2026-invest-on-matrix-wp](../raw/stego/HGAME2026-invest-on-matrix-wp.md)
- [D3CTF2025-d3rpg-signin-wp](../raw/stego/D3CTF2025-d3rpg-signin-wp.md)
- [VNCTF2026-mymnemonic-wp](../raw/crypto/VNCTF2026-mymnemonic-wp.md)
