---
type: family
tags: [crypto, family, encoding, qr, esolang, unicode]
skills: [ctf-crypto]
raw:
  - ../raw/crypto/encodings-qr-and-esolangs.md
  - ../raw/crypto/ACTF2026-special-day-wp.md
  - ../raw/crypto/HGAME2026-打好基础-wp.md
  - ../raw/stego/SUCTF2026-Artifact_OnlineWP.md
updated: 2026-07-27
---

# Encodings, QR and Esolangs

## 作用边界

本页是 Crypto 下的表示层编码 family，用于判断 Base/hex/URL/ROT、UTF/Unicode、IEEE-754/BCD、自定义码表、二维码载荷、esolang 和多层可逆编码链。它的价值是快速确认“决定性障碍是不是编码/表示转换”，以及失败后转向图像、文件格式、pyjail、Reverse 或约束求解。

如果题目核心已经是复杂文件结构、压缩包恢复、图像位平面、沙箱逃逸或交互 oracle，不要继续在编码页里循环尝试。

## 识别信号

- 输入输出主要是字符串、数字列、二维码碎片、Unicode 异常字符、看似程序的符号串、Whitespace/Brainfuck/Piet/Malbolge 等 esolang。
- 字符集、长度、padding、大小写、分隔符、重复模式或可打印率明显指向可逆编码。
- 题面给出“基础”“层层”“乱码”“二维码碎片”“空白”“浮点数”“助记词”等提示。

## 最小证据

- 记录原始样本、长度、字符集、是否有 padding、是否存在分隔符或固定块长。
- 每解一层都保存中间值，并确认中间值的格式证据，而不是凭“看起来像”继续套工具。
- 对二维码，先确认尺寸、finder pattern、version、ECC/mask、chunk 顺序和是否需要图像修复。
- 对 esolang，先确认语言或 opcode 映射，再执行解释器；不要直接把所有符号串当 Brainfuck。

## 首轮路由

| 证据形态 | 先做什么 | 下一跳 |
|---|---|---|
| Base64/Base32/hex/URL/ROT/Caesar | 先按字符集和 padding 验证候选，再逐层记录中间输出。 | [file-triage-archives-and-one-liners.md](file-triage-archives-and-one-liners.md) |
| UTF-16 endian、Unicode tags、variation selector、homoglyph | 保留原始 Unicode code point，先做规范化/隐藏字符提取，再判断是否是文本隐写。 | [exotic-encodings-and-file-formats.md](exotic-encodings-and-file-formats.md) |
| IEEE-754、BCD、数字列、频率映射 | 先确认块长、端序和数值范围，再转 bytes 或频率 rank。 | [oracles-recurrences-captcha-polyglots.md](oracles-recurrences-captcha-polyglots.md) |
| QR 损坏、碎片、目录索引、finder 缺失 | 先恢复网格和 finder pattern，再用 known prefix/ECC 修正。 | [image-bitplane-qr-and-jpeg-stego.md](image-bitplane-qr-and-jpeg-stego.md) |
| Whitespace/Brainfuck/Piet/Malbolge/自定义 esolang | 识别语言和 opcode 映射，先跑小样本验证解释器方向，再执行完整链。 | [vm-z3-sandbox-and-game-basics.md](vm-z3-sandbox-and-game-basics.md) |
| 多层编码链 | 每层只在有格式证据时推进；中间结果若变成文件头、图片、压缩包或脚本，立即 pivot。 | [cross-domain-forensics-technique-map.md](cross-domain-forensics-technique-map.md) |

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [ACTF2026-special-day-wp](../raw/crypto/ACTF2026-special-day-wp.md) | 单层 Base64 解码后还要按题面文本规则规范化 flag body；这类签到题重点是记录输入形态和变换规则。 |
| [HGAME2026-打好基础-wp](../raw/crypto/HGAME2026-打好基础-wp.md) | 大段 emoji 优先考虑 base100；每层解码后用字符集、padding 和可打印率确认下一层 Base 编码，而不是盲目爆破。 |
| [SUCTF2026-Artifact_OnlineWP](../raw/stego/SUCTF2026-Artifact_OnlineWP.md) | 本库按 Stego 边界案例归档：符文文本掩饰字符映射，六面 cube 的空间状态再决定可选命令；完成 rune/plain 映射后应转游戏状态 family。 |
| [D3CTF2021-shellgen2-wp](../raw/web/D3CTF2021-shellgen2-wp.md) | 无字母数字 PHP 生成器本质是受限字符表达式构造，先建字符索引表和递增优化。 |
| [Xp0intCTF2017-MaybeNotStandrad-wp](../raw/reverse/Xp0intCTF2017-MaybeNotStandrad-wp.md) | 输入 45 字节、输出 60 字符且有 64 字符表，是标准 Base64 结构加非标准字母表；先还原表再解码。 |
| [NCTF2026-vm-encryptor-wp](../raw/reverse/NCTF2026-vm-encryptor-wp.md) | 先写自定义 VM disassembler 理清 opcode；真实算法是循环位移/XOR 后进魔改 Base64，再整体 XOR。 |
| [0xGame2020-week1-lowerBase64-wp](../raw/crypto/0xGame2020-week1-lowerBase64-wp.md) | 按 Base64 四字符分组枚举丢失的大小写位，再用已知明文字符集剪枝；字符串完全小写但形态像 Base64，且直接解码得到乱码或不符合 flag 格式。 |
| [0xGame2025-week2-这个b64不太对啊-wp](../raw/crypto/0xGame2025-week2-这个b64不太对啊-wp.md) | 利用 Base64 第四索引只依赖第三明文字节低 6 位的性质构造映射 oracle；编码位拆分保持标准 Base64，只随机置换输出字符表，且允许选择明文查询。 |
| [D3CTF2024-Baldurs-Gate-3-Complete-Spell-List-wp](../raw/crypto/D3CTF2024-Baldurs-Gate-3-Complete-Spell-List-wp.md) | 本题的关键是从标题和数据形态构造码表。九个法术等级在统一减一后自然覆盖 $0$ 至 $8$，而三个九进制位的范围为 $0$ 至 $728$，足以编码 ASCII。先用开头几组验证出 `http`，再批量解码全部数据，可以避免在错误编码方向上继续尝试。 |
| [MoeCTF2024-拼图糕手-revenge-wp](../raw/crypto/MoeCTF2024-拼图糕手-revenge-wp.md) | 这题最容易错在变换顺序。数字没有在原位置编码，而是集中追加后再随整串反转；解密数字链时也必须从已知末位向前递推。二维码只负责承载密文，自定义编码才是决定最终 flag 的步骤。 |
| [UMDCTF2020-halls-harmony-wp](../raw/crypto/UMDCTF2020-halls-harmony-wp.md) | 音频在本题中只是七进制数字的物理载体。先按固定符号时长恢复离散音高，再按三位一组解释为七进制，比直接尝试语音识别或把频率当 ASCII 更符合生成机制。 |
| [UMDCTF2021-whose-base-is-it-anyway-wp](../raw/crypto/UMDCTF2021-whose-base-is-it-anyway-wp.md) | “Base-N”不一定是标准库里的 Base16、Base32 或 Base64。本题的 `n` 表示每个输出符号承载的位数，核心是位流重组。逆向自定义编码时应先记录切片宽度、位序、字符表和尾部剩余位的处理，再严格反转层次顺序。 |

## Technique 下一跳

| 首轮判断 | 具体 technique |
|---|---|
| Base/hex/URL/ROT/字符集或自定义码表形成可逆链 | [layered-encoding-and-symbol-mapping-recovery.md](layered-encoding-and-symbol-mapping-recovery.md) |
| QR/条码/网格符号被分片、旋转、反色或多层叠加 | [qr-and-structured-symbol-reassembly.md](qr-and-structured-symbol-reassembly.md) |
| 扩展名、magic、容器边界或内嵌载荷不一致 | [file-format-and-embedded-payload-identification.md](file-format-and-embedded-payload-identification.md) |

## 合并与拆分结论

- 保留为 `family`：raw 覆盖通用编码、二维码、esolang、Unicode、浮点/数字列等多条路线，第一步判断不同，不能作为单一 technique。
- 已把可逆编码链、QR 结构重组和文件载荷识别拆为 technique；esolang 仍按解释器行为转 reverse family。
- 不并入 [exotic-encodings-and-file-formats.md](exotic-encodings-and-file-formats.md)：后者更偏异常文件/格式清洗，本页负责轻量可逆编码和文本/QR/esolang 首轮判断。

## 常见陷阱

- 用 CyberChef 一路爆破但不记录中间证据，最后无法复现。
- Base64 padding 不对就放弃，没尝试 URL-safe、缺 padding、换行或多层转义。
- Unicode 题先复制到不保留原始 code point 的编辑器，破坏隐藏字符。
- QR 修复只看视觉图，没确认 finder、timing、mask 和 ECC。
- esolang 链执行错方向，未先用小样本验证 opcode 映射。

## 关联技巧

- [crypto-parameter-triage-family.md](crypto-parameter-triage-family.md)
- [cross-category-triage-family.md](cross-category-triage-family.md)
- [file-triage-archives-and-one-liners.md](file-triage-archives-and-one-liners.md)
- [exotic-encodings-and-file-formats.md](exotic-encodings-and-file-formats.md)
- [image-bitplane-qr-and-jpeg-stego.md](image-bitplane-qr-and-jpeg-stego.md)
- [pdf-png-gif-and-text-stego.md](pdf-png-gif-and-text-stego.md)
- [vm-z3-sandbox-and-game-basics.md](vm-z3-sandbox-and-game-basics.md)
- [oracles-recurrences-captcha-polyglots.md](oracles-recurrences-captcha-polyglots.md)

## 原始资料

- [encodings-qr-and-esolangs.md](../raw/crypto/encodings-qr-and-esolangs.md)
- [ACTF2026-special-day-wp](../raw/crypto/ACTF2026-special-day-wp.md)
- [HGAME2026-打好基础-wp](../raw/crypto/HGAME2026-打好基础-wp.md)
- [SUCTF2026-Artifact_OnlineWP](../raw/stego/SUCTF2026-Artifact_OnlineWP.md)
