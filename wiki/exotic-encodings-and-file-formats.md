---
type: family
tags: [cross-category, family, encoding, file-format, unicode, barcode, signal, parser]
skills: [ctf-crypto, ctf-forensics, ctf-reverse, ctf-pwn]
raw:
  - ../raw/crypto/encodings-qr-and-esolangs.md
  - ../raw/crypto/classical-xor-and-substitution-ciphers.md
  - ../raw/reverse/android-games-hardware-and-runtime-platforms.md
  - ../raw/forensics/network-covert-auth-and-reassembly.md
  - ../raw/stego/video-document-and-media-stego.md
  - ../raw/stego/audio-frequency-and-archive-stego.md
  - ../raw/stego/image-bitplane-qr-and-jpeg-stego.md
  - ../raw/pwn/oob-jit-parser-primitives.md
updated: 2026-07-11
---

# Exotic Encodings and File Formats

## 作用边界

本页是异常编码、低频文件格式和结构化数据清洗的跨方向 family 页。它处理的不是普通 Base/hex/ROT，而是 Verilog/HDL、Gray code、二叉树编码、RTF 自定义 tag、SMS PDU、UTF-9、像素颜色编码、MaxiCode、DTMF、音乐音程、二进制网格和语言运行时格式滥用等边界信号。

它与 [encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md) 的关系是上下游：普通可逆编码、QR、Unicode 首检先看编码 family；一旦证据指向特定文件、协议、信号、HDL 或非标准解析器，就在本页判断应转 Crypto、Stego、Forensics、Reverse 还是 Pwn。

## 识别信号

- 数据看似文本，但包含 RTF 控制字、PDU header、UTF-9、Unicode 变体、固定宽度数字、HDL 逻辑或非标准 barcode。
- 图片/音频不是隐写算法本身，而是颜色、音调、音程、网格或条码格式承载比特。
- 解码需要先恢复结构字段、排序、端序、块长、版本或格式规范，而不是直接 CyberChef 多层尝试。
- 出现语言/库解析差异，例如 Ruby `String#unpack` 格式字符串导致的越界读取。

## 最小证据

- 原始字节和格式边界：文件头、控制字、字段长度、块序号、校验、版本、行列尺寸或采样单位。
- 能解释一个小样本如何从格式字段变成 bit/byte/text。
- 如果使用在线或外部解码器，记录输入前处理步骤和可替代本地验证方式。
- 能判断失败是格式识别错、字段排序错、端序错、采样错，还是需要转图像/音频/forensics 页面。

## 首轮路由

| 证据形态 | 首轮判断 | 下一跳 |
|---|---|---|
| Base/hex/ROT/普通 Unicode/QR 首检 | 先在轻量编码 family 中确认不是常规可逆链 | [encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md) |
| Verilog/HDL、逻辑门、状态机 | 若目标是恢复 HDL 行为，先按位宽、时钟边沿和赋值语义建立可运行模型 | [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md) |
| Gray code、二叉树路径、UTF-9、TOPKEK、MaxiCode | 信息存在性公开，先确定格式、码表、遍历顺序和偏移，再做逆映射 | [encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md) |
| RTF unknown destination、自定义 tag | 查看器忽略的字段承载隐藏数据，保留原始控制字并按编号重组 | [video-document-and-media-stego.md](video-document-and-media-stego.md) |
| SMS PDU、UDH、分片消息 | 按 TPDU/UDH 字段解析并按序号重组证据流 | [network-covert-auth-and-reassembly.md](network-covert-auth-and-reassembly.md) |
| 像素行直接映射字符、十六进制数独、普通条码 | 载体只是显式编码，按尺寸、码表和约束恢复 | [encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md) |
| 二进制网格需渲染、QR 块需重排、像素位平面藏第二载荷 | 先恢复隐藏矩阵、极性、方向和模块结构 | [image-bitplane-qr-and-jpeg-stego.md](image-bitplane-qr-and-jpeg-stego.md) |
| DTMF 数字再映射多按键字符 | 保留停顿分组；音频提取完成后按公开键盘码表逆映射 | [encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md) |
| 音符/音程隐蔽承载 nibble | 先提取音符序列，再用 known prefix 校准音阶映射 | [audio-frequency-and-archive-stego.md](audio-frequency-and-archive-stego.md) |
| Ruby `String#unpack` 或类似解析器越界 | 确认攻击者是否控制格式字符串、运行时是否受影响以及能否形成读取原语 | [oob-jit-parser-primitives-family.md](oob-jit-parser-primitives-family.md) |

## 合并与拆分结论

原始混合综述已经按决定性主障碍拆入 Crypto、Stego、Forensics、Reverse 和 Pwn 的现有 raw 资料卷。本页继续保留为 family，不与 [encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md) 合并：编码页负责可逆表示本身，本页负责从异常载体或格式信号判断下一方向。这里保留的是跨方向 pivot，而不是物理兜底分类。

## 常见陷阱

- 把低频格式当普通乱码，反复套 Base/ROT 而不看字段结构。
- 复制 Unicode 或 RTF 内容时丢失控制字、空格、反斜杠或 code point。
- 图像/条码类只看视觉效果，没确认模块尺寸、坐标系和采样方式。
- DTMF/音乐题不先做频谱或音符序列，直接按听感猜字符。
- 使用在线解码器后不记录前处理，导致无法复现。

## 关联技巧

- [encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md)
- [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md)
- [network-covert-auth-and-reassembly.md](network-covert-auth-and-reassembly.md)
- [video-document-and-media-stego.md](video-document-and-media-stego.md)
- [image-bitplane-qr-and-jpeg-stego.md](image-bitplane-qr-and-jpeg-stego.md)
- [audio-frequency-and-archive-stego.md](audio-frequency-and-archive-stego.md)
- [oob-jit-parser-primitives-family.md](oob-jit-parser-primitives-family.md)

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [NCTF2026-what-a-mess-wp](../raw/stego/NCTF2026-what-a-mess-wp.md) | 本库按 Stego 边界案例归档：Unicode、全角、零宽字符和格式变体用于掩饰记录语义；首轮仍应先做规范化、白名单和校验位统计。 |

## 原始资料

- 编码与显式格式：[encodings-qr-and-esolangs.md](../raw/crypto/encodings-qr-and-esolangs.md)
- known-prefix XOR：[classical-xor-and-substitution-ciphers.md](../raw/crypto/classical-xor-and-substitution-ciphers.md)
- Verilog/HDL：[android-games-hardware-and-runtime-platforms.md](../raw/reverse/android-games-hardware-and-runtime-platforms.md)
- SMS PDU：[network-covert-auth-and-reassembly.md](../raw/forensics/network-covert-auth-and-reassembly.md)
- RTF 文档隐写：[video-document-and-media-stego.md](../raw/stego/video-document-and-media-stego.md)
- 音频与音乐音程：[audio-frequency-and-archive-stego.md](../raw/stego/audio-frequency-and-archive-stego.md)
- 二进制网格与 QR 恢复：[image-bitplane-qr-and-jpeg-stego.md](../raw/stego/image-bitplane-qr-and-jpeg-stego.md)
- Ruby `String#unpack` 越界读取：[oob-jit-parser-primitives.md](../raw/pwn/oob-jit-parser-primitives.md)
