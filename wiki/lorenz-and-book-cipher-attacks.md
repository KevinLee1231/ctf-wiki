---
type: technique
tags: [crypto, technique, lorenz, book-cipher, historical-cipher, baudot, known-plaintext]
skills: [ctf-crypto]
raw:
  - ../raw/crypto/lorenz-and-book-cipher-attacks.md
updated: 2026-07-06
---

# Lorenz and Book Cipher Attacks

## 适用场景

题目不是常规现代密码原语，而是历史机械密码、书本/文本位置密码或字符集受限的古典变体。关键不是套 AES/RSA 工具，而是恢复编码、轮组/步进规则或参考文本中的位置约束。

## 识别信号

- 题面出现 Lorenz、Tunny、Baudot、ITA2、wheel、chi/psi/mu、book、reference text、distance list。
- 明文/密文是 5-bit 字符、有限字母表或一串位置距离。
- 已知部分明文足以推出 keystream，或者题目给了参考书/文章作为字典。
- 候选结果受 charset 强约束，例如只允许 printable、flag charset 或密码字符集。

## 最小证据

- 能把输入转换成统一符号空间：ITA2/Baudot 5-bit code 或 reference text index。
- Lorenz 题至少能由 known plaintext 得到 keystream，并计算 `delta_k[i]=k[i]^k[i+1]`。
- Book cipher 题至少能枚举一个起点并按 distance list 得到候选字符串。

## 解法骨架

1. 先确认编码层：字母表、shift code、大小写、空格/标点是否保留。
2. Lorenz：known plaintext → keystream → delta keystream；利用 psi 不总是步进的偏置恢复 chi delta。
3. 对每个 chi wheel 积分 delta，保留 0/1 起点歧义，再用明文可读性或已知前缀剪枝。
4. 从剩余 delta 推 psi stepping pattern，再恢复 mu/psi 轮或暴力剩余相位。
5. Book cipher：枚举起点，按距离前进取字符；用 valid charset 和 flag 格式快速过滤。

## 历史密码分支

| 结构 | 判断方式 |
|---|---|
| Lorenz delta attack | `delta_k = delta_chi XOR delta_psi`，psi 静止比例高，delta 统计偏向 chi。 |
| ITA2/Baudot | 先处理 FIGS/LTRS shift，否则解出的字符会错位。 |
| Book cipher distance | 起点比距离更关键；charset 约束能把数万候选压到少量。 |
| Known plaintext | 历史密码题常靠已知前缀/格式做相位和轮位确认。 |

## 常见陷阱

- 把历史密码当普通 XOR 频率题，忽略轮组步进结构。
- 没处理 Baudot shift，导致明文看似随机。
- Book cipher 忽略参考文本预处理差异：大小写、换行、空格、标点都会改变位置。
- 候选太多时不先用 flag charset/known prefix 过滤。

## 关联技巧

- [classical-xor-and-substitution-ciphers.md](classical-xor-and-substitution-ciphers.md)
- [rc4-lfsr-and-keystream-reuse.md](rc4-lfsr-and-keystream-reuse.md)
- [encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md)
- [exotic-encodings-and-file-formats.md](exotic-encodings-and-file-formats.md)
- [crypto-parameter-triage-family.md](crypto-parameter-triage-family.md)

## 原始资料

- [lorenz-and-book-cipher-attacks.md](../raw/crypto/lorenz-and-book-cipher-attacks.md)
