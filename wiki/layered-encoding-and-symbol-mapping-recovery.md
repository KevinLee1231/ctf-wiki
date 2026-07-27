---
type: technique
tags: [crypto, encoding, custom-alphabet, base, representation]
skills: [ctf-crypto]
raw:
  - ../raw/crypto/encodings-qr-and-esolangs.md
  - ../raw/crypto/classical-xor-and-substitution-ciphers.md
updated: 2026-07-27
---

# Layered Encoding and Symbol-Mapping Recovery

## 适用场景

秘密经过 Base/hex/URL/ROT、字符集转换、自定义码表、位序重排或多层可逆表示变换；决定性障碍是恢复变换顺序和数据边界，而非破解密码密钥。

## 识别信号

- 字符集、长度和 padding 符合某种编码，但单层解码后仍是结构化文本。
- 出现固定大小字母表、频率映射、键盘布局、符号表或位组重排。
- 解码中间层逐步显露 magic、自然语言或下一层编码特征。

## 最小证据

- 记录原始字节、可见字符集、长度模数和 padding。
- 每层都给出可逆变换及输出类型，不依赖“看起来像”。
- 最终结果应通过 magic、校验、语法或 flag 格式验证。

## 解法骨架

1. 从熵、字符集和长度约束识别候选表示层。
2. 每次只应用一个变换，保留中间产物和反向重编码结果。
3. 自定义码表先恢复 symbol-to-value，再处理 bit packing、端序和字符集。
4. 用 beam/小规模搜索处理层序不确定性，但以结构校验剪枝。

## 关键变体

- 标准编码链：Base、hex、URL、ROT 和压缩层交替。
- 自定义 alphabet：符号值可由频率、已知前缀或码表泄露确定。
- 位/字节重排：需区分 bit order、endianness 和分组宽度。

## 常见陷阱

- 只因输出可打印就停止解码。
- 丢弃前导零或错误处理 Base padding。
- 同时尝试过多变换，无法解释哪一步提供有效证据。

## 关联技巧

- [encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md)
- [exotic-encodings-and-file-formats.md](exotic-encodings-and-file-formats.md)
- [qr-and-structured-symbol-reassembly.md](qr-and-structured-symbol-reassembly.md)
- [file-format-and-embedded-payload-identification.md](file-format-and-embedded-payload-identification.md)

## 原始资料

- [encodings-qr-and-esolangs.md](../raw/crypto/encodings-qr-and-esolangs.md)
- [classical-xor-and-substitution-ciphers.md](../raw/crypto/classical-xor-and-substitution-ciphers.md)
