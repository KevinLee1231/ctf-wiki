---
type: family
tags: [crypto, family, hash, mac, oracle, protocol]
skills: [ctf-crypto]
raw:
  - ../raw/crypto/hash-protocol-and-oracle-attacks.md
updated: 2026-06-12
---

# Hash, Protocol and Oracle Attacks

## 作用边界

本页是 hash / MAC / 协议 oracle family。共同模式是：攻击者不能直接拿到 key 或明文，但能利用 hash 状态可扩展、MAC/CRC 线性、压缩长度、padding/解析错误、时间差、协议公钥校验缺失或会话反馈，把隐藏状态逐步恢复或伪造。

分组模式本身的字段拆分先看 [block-mode-misuse-family.md](block-mode-misuse-family.md)；RSA 专用 oracle 先看 [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md)。

## 变体路由

| 信号 | 先确认 | 下一跳 |
|---|---|---|
| MD5/SHA1 length extension、secret-prefix MAC | 服务端是否用 `hash(secret || msg)`，是否可控追加数据和原始长度 | 本页 raw；工具入口见 [crypto-tooling.md](crypto-tooling.md) |
| CRC/线性 MAC/XOR aggregate/hash basis | tag 是否可写成 GF(2) 线性关系或可组合状态 | [classical-xor-and-substitution-ciphers.md](classical-xor-and-substitution-ciphers.md)、[number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md) |
| Compression oracle / CRIME-style | secret 与可控前缀同包压缩，响应长度稳定可测 | 本页 raw |
| Padding/CBC/UTF-8/JSON parse oracle | 错误文本、状态码、时间或业务状态能区分候选 | [block-mode-misuse-family.md](block-mode-misuse-family.md) |
| SRP/DH 公钥校验缺失、协议 proof 可预测 | 发送特殊公钥后 shared secret 是否固定为 0 或小集合 | [auth-edge-cases-and-protocol-bypasses.md](auth-edge-cases-and-protocol-bypasses.md) |
| Rabin/RSA LSB/noisy oracle | 每轮查询是否稳定给出一位、区间或带噪反馈 | [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md)、[number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md) |
| Hash state reversal、cycle detection、sponge MITM | 状态转移是否可逆、周期可测或可 meet-in-the-middle | 本页 raw |

## 合并与拆分结论

- 保留为 family：raw 横跨 hash、MAC、block mode、协议和 oracle，统一价值是判断“可观测差异如何转成隐藏状态”。
- 不与 block-mode family 合并：block-mode 页关注字段和模式误用；本页关注 oracle、hash/MAC 状态和协议反馈。
- 不拆 length-extension / compression / CRC 小页：当前多为短案例速查，后续有多篇 WP 支撑时再拆具体 technique。

## 常见误判

- 有 hash 就直接爆破 key，没先看是否 length extension、线性组合或状态泄露。
- Oracle 只用一次响应判断，没确认响应差异可重复且与候选字节单调相关。
- 把 padding error、MAC error、业务 error 混在一起，导致脚本判断条件不稳定。
- 协议绕过只看数学式，不复现消息编码、HMAC proof、session 绑定和服务端检查顺序。

## 关联页面

- [crypto-parameter-triage-family.md](crypto-parameter-triage-family.md)
- [block-mode-misuse-family.md](block-mode-misuse-family.md)
- [classical-xor-and-substitution-ciphers.md](classical-xor-and-substitution-ciphers.md)
- [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md)
- [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md)
- [crypto-tooling.md](crypto-tooling.md)

## 原始资料

- [hash-protocol-and-oracle-attacks.md](../raw/crypto/hash-protocol-and-oracle-attacks.md)
