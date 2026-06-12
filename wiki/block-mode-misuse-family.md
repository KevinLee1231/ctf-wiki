---
type: family
tags: [crypto, family]
skills: [ctf-crypto]
raw:
  - ../raw/crypto/aes-modes-mac-and-oracles.md
updated: 2026-05-22
---

# 分组密码模式、MAC 与 Oracle 误用技巧族

## 适用场景

本页是 `分组密码模式、MAC 与 Oracle 误用技巧族` 技巧家族页，用来承接多个相邻技巧和案例；先用于判断是否属于这一族，再选择具体变体。

本页不是 raw 的目录页；它把原始资料中的案例压缩成可迁移的判断信号、最小证据和解题骨架。

## 识别信号

- 题目给出可查询加密/解密/校验接口，或给出 nonce、IV、tag、MAC、错误信息和多组密文。
- 同一 key 下出现重复 nonce/IV/counter，或 IV、nonce、counter 可控、固定、可预测。
- 修改密文后响应差异稳定：padding error、MAC error、JSON parse error、权限变化、明文格式变化。
- 认证与加密边界不清：encrypt-then-MAC / MAC-then-encrypt / raw hash / CRC / 线性 MAC 混用。
- 明文结构已知或可猜：flag 前缀、JSON、PNG/PDF header、固定字段、block boundary。

## 最小证据

- 至少收集两组同 key 下的输入输出，能判断 mode、block size、nonce/IV/counter/tag 的位置。
- 有一个最小可重放请求或脚本，能稳定区分“合法 / 非法 / 格式错误 / padding 错误 / MAC 错误”。
- 明确攻击目标：恢复明文、伪造密文、伪造 tag、恢复 key stream，或绕过认证。
- 如果走 oracle，必须量化 oracle 信号：状态码、错误文本、时间、长度、业务状态或返回对象。

## 解法骨架

1. 切分协议字段：`nonce/iv | ciphertext | tag/mac | aad | payload`，不要把整包直接丢给工具。
2. 先用 block size、重复块、长度变化、错误差异判断模式和认证顺序。
3. 根据证据选择最小攻击：padding oracle、CTR/GCM nonce reuse、CBC bitflip、ECB 结构泄漏、MAC 长度扩展或线性伪造。
4. 写最小脚本验证一个字节、一个块或一个字段能否被控制。
5. 扩展到完整明文恢复或目标字段伪造；最后用服务端响应或本地复现做正向验证。

## 关键变体

| 变体 | 优先证据 | 下一跳页面 | 失败后 pivot |
|---|---|---|---|
| CBC padding oracle | padding/MAC/parse 错误可区分，block size 稳定 | [hash-protocol-and-oracle-attacks.md](hash-protocol-and-oracle-attacks.md) | 若错误不可区分，改查 timing、长度或业务状态 oracle。 |
| CBC bitflip / block boundary | 前一块可控，明文有固定字段 | [classical-xor-and-substitution-ciphers.md](classical-xor-and-substitution-ciphers.md) | 若有认证 tag，先判断 MAC 顺序或签名绕过。 |
| CTR/GCM nonce reuse | nonce/counter 重复，同 key 多密文 | [rc4-lfsr-and-keystream-reuse.md](rc4-lfsr-and-keystream-reuse.md) | 如果只有 tag 可查，转 GHASH/forbidden attack 或协议 oracle。 |
| ECB 图像/结构泄漏 | 重复明文块对应重复密文块 | [file-signatures-and-flag-artifact-hunting.md](file-signatures-and-flag-artifact-hunting.md) | 如果块不重复，检查压缩、随机 padding 或分组边界。 |
| 线性 MAC / CRC 伪造 | tag 满足 XOR/GF(2)/CRC 可组合关系 | [hash-protocol-and-oracle-attacks.md](hash-protocol-and-oracle-attacks.md) | 若非线性，回到长度扩展、keyed hash 或签名实现。 |
| PDF / HashClash chosen-prefix | 需要构造两个同 hash 但语义不同文件 | [crypto-tooling.md](crypto-tooling.md) | 若服务端二次解析文件，联合 Web/parser differential 页面。 |

## 常见陷阱

- 看到 AES 就直接猜 CBC；应先用字段长度、重复块和错误差异确认模式。
- 把“解密失败”和“业务失败”混在一起；oracle 的价值来自可稳定区分的最小信号。
- GCM nonce reuse 不等于只 XOR 明文；伪造 tag 还要处理 GHASH 关系。
- 只在本地复现 crypto，不复现服务端封包、编码、base64/urlencode 和 JSON 规范化。
- 忽略格式已知明文；flag 前缀、文件头和 JSON key 常常足够恢复 keystream。


## 关联技巧

- [classical-xor-and-substitution-ciphers.md](classical-xor-and-substitution-ciphers.md)
- [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md)
- [ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md)
- [embedded-python-pyd-custom-aes.md](embedded-python-pyd-custom-aes.md)
- [exotic-secret-sharing-rabin-and-polynomials.md](exotic-secret-sharing-rabin-and-polynomials.md)
- [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md)
- [rsa-attacks.md](rsa-attacks.md)
- [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md)
- [lattice-and-lwe.md](lattice-and-lwe.md)
- [prng-z3-lcg-and-timing-attacks.md](prng-z3-lcg-and-timing-attacks.md)
- [mt-lcg-and-seed-recovery.md](mt-lcg-and-seed-recovery.md)
- [homomorphic-and-exotic-algebra.md](homomorphic-and-exotic-algebra.md)
- [zkp-secret-sharing-and-proof-systems.md](zkp-secret-sharing-and-proof-systems.md)
- [lorenz-and-book-cipher-attacks.md](lorenz-and-book-cipher-attacks.md)

## 原始资料

- [aes-modes-mac-and-oracles.md](../raw/crypto/aes-modes-mac-and-oracles.md)
