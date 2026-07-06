---
type: family
tags: [misc, family, encoding, file-format, unicode, barcode, signal]
skills: [ctf-misc, ctf-forensics]
raw:
  - ../raw/misc/exotic-encodings-and-file-formats.md
updated: 2026-06-12
---

# Exotic Encodings and File Formats

## 作用边界

本页是异常编码、低频文件格式和结构化数据清洗 family 页。它处理的不是普通 Base/hex/ROT，而是 Verilog/HDL、Gray code、二叉树编码、RTF 自定义 tag、SMS PDU、UTF-9、像素颜色编码、MaxiCode、DTMF、音乐音程、二进制网格和语言运行时格式滥用等边界题。

它与 [encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md) 的关系是上下游：普通可逆编码、QR、Unicode 首检先看编码 family；一旦证据指向特定文件/协议/信号格式或非标准解析器，就转到本页。

## 识别信号

- 数据看似文本，但包含 RTF 控制字、PDU header、UTF-9、Unicode 变体、固定宽度数字、HDL 逻辑或非标准 barcode。
- 图片/音频不是隐写算法本身，而是颜色、音调、音程、网格或条码格式承载比特。
- 解码需要先恢复结构字段、排序、端序、块长、版本或格式规范，而不是直接 CyberChef 多层尝试。
- 出现语言/库解析差异，例如 Ruby `Array#unpack` 这类格式字符串导致的越界读取。

## 最小证据

- 原始字节和格式边界：文件头、控制字、字段长度、块序号、校验、版本、行列尺寸或采样单位。
- 能解释一个小样本如何从格式字段变成 bit/byte/text。
- 如果使用在线或外部解码器，记录输入前处理步骤和可替代本地验证方式。
- 能判断失败是格式识别错、字段排序错、端序错、采样错，还是需要转图像/音频/forensics 页面。

## 首轮路由

| 证据形态 | 首轮判断 | 下一跳 |
|---|---|---|
| Base/hex/ROT/普通 Unicode/QR 首检 | 先在轻量编码 family 中确认不是常规可逆链 | [encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md) |
| Verilog/HDL、逻辑门、状态机 | 先把组合/时序逻辑翻译成可跑模型，再从输入输出关系恢复 bit | [vm-z3-sandbox-and-game-basics.md](vm-z3-sandbox-and-game-basics.md) |
| Gray code、二叉树路径、TOPKEK 这类自定义映射 | 先确定码表、遍历顺序和旋转/偏移，再做逆映射 | [oracles-recurrences-captcha-polyglots.md](oracles-recurrences-captcha-polyglots.md) |
| RTF tag、SMS PDU、UTF-9、结构化文本格式 | 先按规范拆字段和排序，再提取 payload 或嵌入文件 | [file-triage-archives-and-one-liners.md](file-triage-archives-and-one-liners.md) |
| 像素颜色编码、二进制网格、QR 拼装、MaxiCode | 先恢复网格尺寸、模块形状和坐标，再转图像/条码解码 | [image-bitplane-qr-and-jpeg-stego.md](image-bitplane-qr-and-jpeg-stego.md) |
| DTMF、多按键键盘、音乐音程 | 先提取频率/音符序列，再转键盘映射或音程编码 | [audio-frequency-and-archive-stego.md](audio-frequency-and-archive-stego.md) |
| Ruby `Array#unpack` 或库解析差异 | 先确认格式字符串语义和越界读取边界，再判断是否转 Web/Pwn/Reverse | [parser-wrapper-and-legacy-ssrf-tricks.md](parser-wrapper-and-legacy-ssrf-tricks.md), [interpreter-jit-canary-and-integer-exploits.md](interpreter-jit-canary-and-integer-exploits.md) |

## 合并与拆分结论

本页应保留为 family，不并入 [encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md)。后者负责轻量编码和 QR/esolang 首轮判断，本页负责需要格式规范、字段清洗、信号映射或解析器语义的低频格式。当前也不拆小页，因为每个格式 raw 数量不足，拆分会只剩孤立案例。

## 常见陷阱

- 把低频格式当普通乱码，反复套 Base/ROT 而不看字段结构。
- 复制 Unicode 或 RTF 内容时丢失控制字、空格、反斜杠或 code point。
- 图像/条码类只看视觉效果，没确认模块尺寸、坐标系和采样方式。
- DTMF/音乐题不先做频谱或音符序列，直接按听感猜字符。
- 使用在线解码器后不记录前处理，导致无法复现。

## 关联技巧

- [encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md)
- [file-triage-archives-and-one-liners.md](file-triage-archives-and-one-liners.md)
- [image-bitplane-qr-and-jpeg-stego.md](image-bitplane-qr-and-jpeg-stego.md)
- [audio-frequency-and-archive-stego.md](audio-frequency-and-archive-stego.md)
- [oracles-recurrences-captcha-polyglots.md](oracles-recurrences-captcha-polyglots.md)
- [vm-z3-sandbox-and-game-basics.md](vm-z3-sandbox-and-game-basics.md)
- [parser-wrapper-and-legacy-ssrf-tricks.md](parser-wrapper-and-legacy-ssrf-tricks.md)
- [misc-tooling.md](misc-tooling.md)

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [NCTF2026-what-a-mess-wp](../raw/misc/NCTF2026-what-a-mess-wp.md) | 表格字段被 Unicode、全角、零宽字符和格式变体污染，先做规范化、白名单和校验位统计。 |

## 原始资料

- [exotic-encodings-and-file-formats.md](../raw/misc/exotic-encodings-and-file-formats.md)
