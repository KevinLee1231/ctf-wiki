---
type: family
tags: [crypto, family, stream-cipher, rc4, lfsr, keystream]
skills: [ctf-crypto]
raw:
  - ../raw/crypto/rc4-lfsr-and-keystream-reuse.md
updated: 2026-06-12
---

# RC4, LFSR and Keystream Reuse

## 适用场景

题目表现为流密码、LFSR、RC4、XOR keystream 或自定义递推产生的逐字节/逐 bit 加密。重点不是恢复 seed，而是从已知明文、重复 keystream、统计偏差或线性结构中恢复 keystream、反馈多项式、初始状态或明文。

## 识别信号

- 密文与明文长度一致，没有分组 padding，或加密逻辑是逐字节 XOR/加法。
- 多个密文使用相同 key/nonce/stream，或文件头、flag 前缀、协议固定字段可作为 crib。
- 源码出现 LFSR、tap、feedback polynomial、RC4 KSA/PRGA、Fibonacci/Galois 字样。
- 输出 bit 有线性递推、周期性、自相关峰、RC4 bias 或 DNS/PCAP 泄露的 key 片段。

## 最小证据

- 至少一段可对齐的 known plaintext 或两个共享 keystream 的密文。
- LFSR 位序、移位方向、输出 bit 位置和 Galois/Fibonacci 实现约定。
- 如果依赖 RC4 bias，需要足够多独立样本；单个短密文通常不能靠 bias 解决。
- 能用恢复出的 keystream 或状态正向解出一段明文。

## 解法骨架

1. 先判断是 keystream reuse、线性 LFSR、RC4 bias 还是自定义 stream 约束。
2. 对 known plaintext，先 XOR 出 keystream，再看能否 Berlekamp-Massey、线性方程或 tap recovery。
3. 对多密文 reuse，先做 `C1 xor C2 = P1 xor P2` 和 crib dragging。
4. 对非线性过滤或折叠输出，转 [prng-z3-lcg-and-timing-attacks.md](prng-z3-lcg-and-timing-attacks.md) 建约束。

## 关键变体

| 变体 | 触发证据 | 处理路线 |
|---|---|---|
| LFSR known plaintext | 文件头、flag 前缀或协议常量可对齐到密文。 | XOR 得 keystream bit，Berlekamp-Massey 求最短递推，再恢复后续 stream。 |
| Galois/Fibonacci 约定不明 | 源码只给 tap 或多项式，输出顺序不清。 | 同时测试移位方向和输出 bit 约定，用已知明文正向校验。 |
| correlation/filter LFSR | 多个 LFSR 或过滤函数组合输出。 | 先找线性 annihilator、相关峰或低复杂度子状态，必要时用 Z3。 |
| RC4 second-byte bias | 大量独立 RC4 加密样本，目标字节位置固定。 | 统计偏差恢复目标字节；样本不足时 pivot 到 key/nonce 复用。 |
| keystream reuse | 两个以上密文共享 stream 或 nonce。 | 密文互 XOR、crib dragging、文件头推 key，关联 [classical-xor-and-substitution-ciphers.md](classical-xor-and-substitution-ciphers.md)。 |
| custom recurrence stream | 递推不是标准 LFSR/RC4，但输出仍逐字节 XOR。 | 先化为线性/低维状态；不能线性化再转约束求解。 |
| PCAP/DNS 泄露 key | hostname、协议字段或流量重组暴露 key/stream 片段。 | 先转 [network-covert-auth-and-reassembly.md](network-covert-auth-and-reassembly.md) 补齐会话和方向。 |

## 常见陷阱

- 没有对齐 known plaintext，导致 keystream bit 全部错位。
- 把 RC4 bias 当作单样本攻击。
- 忽略位序、端序和 LFSR 输出位置，BM 结果无法复算。
- 遇到重复 XOR 不先做 crib dragging，直接上复杂 solver。

## 关联技巧

- [classical-xor-and-substitution-ciphers.md](classical-xor-and-substitution-ciphers.md)
- [mt-lcg-and-seed-recovery.md](mt-lcg-and-seed-recovery.md)
- [prng-z3-lcg-and-timing-attacks.md](prng-z3-lcg-and-timing-attacks.md)
- [hash-protocol-and-oracle-attacks.md](hash-protocol-and-oracle-attacks.md)
- [network-covert-auth-and-reassembly.md](network-covert-auth-and-reassembly.md)
- [crypto-tooling.md](crypto-tooling.md)

## 原始资料

- [rc4-lfsr-and-keystream-reuse.md](../raw/crypto/rc4-lfsr-and-keystream-reuse.md)
