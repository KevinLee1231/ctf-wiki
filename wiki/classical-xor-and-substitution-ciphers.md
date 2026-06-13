---
type: family
tags: [crypto, family, classical, xor, substitution]
skills: [ctf-crypto, ctf-misc]
raw:
  - ../raw/crypto/classical-xor-and-substitution-ciphers.md
updated: 2026-06-12
---

# Classical XOR and Substitution Ciphers

## 作用边界

本页是轻量古典密码、XOR、替换和已知明文恢复 family。它适合处理 Vigenere、Atbash、Polybius、rotating substitution、many-time pad、cascade XOR、文件头推 key、图像 Caesar、semaphore/photo、book cipher 等“结构简单但识别信号容易分散”的题。

现代分组模式、MAC、nonce reuse 和 oracle 不在本页展开，转 [block-mode-misuse-family.md](block-mode-misuse-family.md) 或 [hash-protocol-and-oracle-attacks.md](hash-protocol-and-oracle-attacks.md)。

## 变体路由

| 信号 | 先确认 | 下一跳 |
|---|---|---|
| Vigenere/Atbash/Polybius/rotating wheel | 字母表、key length、频率、crib、坐标表和是否有变体置换 | 本页 raw |
| 多字节 XOR、cascade XOR、weak XOR verifier | key length、已知明文、频率、首字节可枚举和 forward check | 本页 raw；流密码转 [rc4-lfsr-and-keystream-reuse.md](rc4-lfsr-and-keystream-reuse.md) |
| many-time pad / OTP reuse | 多密文同 key、可猜明文、空格/flag 前缀和 crib dragging | [rc4-lfsr-and-keystream-reuse.md](rc4-lfsr-and-keystream-reuse.md) |
| 文件头/尾推 XOR key | 已知 magic、trailer、块边界和 key 周期 | [file-signatures-and-flag-artifact-hunting.md](file-signatures-and-flag-artifact-hunting.md) |
| book/lorenz/长文本替换 | 外部文本、同步轮、crib 和统计模型 | [lorenz-and-book-cipher-attacks.md](lorenz-and-book-cipher-attacks.md) |
| 图片 Caesar、semaphore、视觉编码 | 可视排列、坐标/条带偏移、手势表或图像通道 | [image-bitplane-qr-and-jpeg-stego.md](image-bitplane-qr-and-jpeg-stego.md)、[misc-cross-category-triage-family.md](misc-cross-category-triage-family.md) |

## 合并与拆分结论

- 保留为 family：它负责轻量 crypto 与 misc 编码题之间的分流，避免把所有 XOR/替换短案例散成低密度 technique。
- 不与 `lorenz-and-book-cipher-attacks.md` 合并：后者更适合作具体历史/书本/长文本 cipher 技巧页。
- 不与 `rc4-lfsr-and-keystream-reuse.md` 合并：本页处理简单 XOR 和古典替换，后者处理可建模 keystream/状态恢复。

## 常见误判

- 一看到 XOR 就只爆破单字节 key，没先估 key length、周期和已知文件头。
- 古典替换没有先固定 alphabet 和分组方式，导致频率分析失真。
- many-time pad 题把每条密文单独解，忽略跨密文 XOR 会消去 keystream。
- 视觉古典题没有记录坐标、条带方向和变换顺序，结果不可复算。

## 关联页面

- [crypto-parameter-triage-family.md](crypto-parameter-triage-family.md)
- [block-mode-misuse-family.md](block-mode-misuse-family.md)
- [hash-protocol-and-oracle-attacks.md](hash-protocol-and-oracle-attacks.md)
- [rc4-lfsr-and-keystream-reuse.md](rc4-lfsr-and-keystream-reuse.md)
- [lorenz-and-book-cipher-attacks.md](lorenz-and-book-cipher-attacks.md)
- [misc-cross-category-triage-family.md](misc-cross-category-triage-family.md)

## 原始资料

- [classical-xor-and-substitution-ciphers.md](../raw/crypto/classical-xor-and-substitution-ciphers.md)
