---
type: technique
tags: [crypto, stream-cipher, rc4, lfsr, keystream-reuse]
skills: [ctf-crypto]
raw:
  - ../raw/crypto/rc4-lfsr-and-keystream-reuse.md
  - ../raw/crypto/classical-xor-and-substitution-ciphers.md
  - ../raw/reverse/D3CTF2021-zigzag-encryptor-wp.md
updated: 2026-07-28
---

# Stream-Cipher Keystream Reuse and State Recovery

## 适用场景

多个明文复用同一 keystream，或 LFSR/RC4/自定义流密码暴露足量输出，可从已知明文、线性递推、偏差或状态约束恢复密钥流与内部状态。

## 识别信号

- 两份等长密文异或后出现自然语言、文件头或可拖拽结构。
- 已知/可猜明文片段能直接给出 keystream。
- 输出满足线性递推，或 RC4 key/nonce 复用、初始化弱化。

## 最小证据

- 确认密文对齐、nonce/IV 与加密起始 offset。
- 对 LFSR 给出输出位顺序、反馈方向和足够线性复杂度。
- 用恢复 keystream 解密至少两个独立位置。

## 解法骨架

1. 规范化字节/比特序和消息 offset。
2. 复用场景先做密文异或和 crib dragging；LFSR 用 Berlekamp-Massey/线性方程。
3. RC4 或自定义状态机按 KSA/PRGA 精确复刻，再结合已知明文约束状态/密钥。
4. 向前后扩展 keystream，并以格式、校验和或多密文一致性验证。

## 关键变体

- Many-time pad：同一 keystream 跨消息复用。
- LFSR combiner：需先分离或利用相关性/低线性复杂度。
- RC4 state/key weakness：重点在 nonce、KSA 和初始输出偏差。

## 常见陷阱

- 密文 offset 未对齐就进行 crib dragging。
- 混淆 bit endianness 与反馈多项式方向。
- 已知明文只在一个位置自洽，未做跨消息验证。

## 关联技巧

- [rc4-lfsr-and-keystream-reuse.md](rc4-lfsr-and-keystream-reuse.md)
- [classical-xor-and-substitution-ciphers.md](classical-xor-and-substitution-ciphers.md)
- [linear-prng-state-and-seed-recovery.md](linear-prng-state-and-seed-recovery.md)
- [lorenz-and-book-cipher-attacks.md](lorenz-and-book-cipher-attacks.md)

## 原始资料

- [rc4-lfsr-and-keystream-reuse.md](../raw/crypto/rc4-lfsr-and-keystream-reuse.md)
- [classical-xor-and-substitution-ciphers.md](../raw/crypto/classical-xor-and-substitution-ciphers.md)
- [D3CTF2021-zigzag-encryptor-wp](../raw/reverse/D3CTF2021-zigzag-encryptor-wp.md)
