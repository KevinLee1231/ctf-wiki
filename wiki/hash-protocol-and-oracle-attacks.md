---
type: family
tags: [crypto, family, hash, mac, oracle, protocol]
skills: [ctf-crypto]
raw:
  - ../raw/crypto/hash-protocol-and-oracle-attacks.md
  - ../raw/crypto/ACTF2026-ohmycaptcha-wp.md
  - ../raw/crypto/RCTF2025-suanhash-wp.md
updated: 2026-07-06
---

# Hash, Protocol and Oracle Attacks

## 作用边界

本页是 hash / MAC / 协议 oracle family。共同模式是：攻击者不能直接拿到 key 或明文，但能利用 hash 状态可扩展、MAC/CRC 线性、压缩长度、padding/解析错误、时间差、协议公钥校验缺失或会话反馈，把隐藏状态逐步恢复或伪造。

分组模式本身的字段拆分先看 [block-mode-misuse-family.md](block-mode-misuse-family.md)；RSA 专用 oracle 先看 [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md)。

## 识别信号

- 服务暴露 hash、MAC、CRC、digest、proof、padding、parse error、长度、时间或协议状态差异。
- 攻击者能控制消息前缀/后缀、协议参数、公钥、密文、表达式输出或查询序列。
- 响应不是直接给明文，而是通过错误类型、长度、布尔结果、会话状态或下一轮输出泄露信息。
- tag/hash 状态可能可扩展、可线性组合、可碰撞、可反推内部差分或可被 oracle 逐步收缩。

## 最小证据

- 保存一个可重放请求，能稳定区分至少两类响应差异。
- 明确 oracle 信号来自哪一层：padding、MAC、JSON/UTF-8 parse、业务状态、压缩长度、时间或协议 proof。
- 对 hash/MAC，要说明消息格式、secret 位置、原始长度假设和 tag 校验顺序。
- 对协议 oracle，要记录会话绑定、编码方式、参数检查顺序和重复查询是否改变状态。

## 变体路由

| 信号 | 先确认 | 下一跳 |
|---|---|---|
| MD5/SHA1 length extension、secret-prefix MAC | 服务端是否用 `hash(secret || msg)`，是否可控追加数据和原始长度 | 本页 raw；工具入口见 [crypto-tooling.md](crypto-tooling.md) |
| CRC/线性 MAC/XOR aggregate/hash basis | tag 是否可写成 GF(2) 线性关系或可组合状态 | [classical-xor-and-substitution-ciphers.md](classical-xor-and-substitution-ciphers.md)、[number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md) |
| Compression oracle / CRIME-style | secret 与可控前缀同包压缩，响应长度稳定可测 | 本页 raw |
| Padding/CBC/UTF-8/JSON parse oracle | 错误文本、状态码、时间或业务状态能区分候选 | [block-mode-misuse-family.md](block-mode-misuse-family.md) |
| SRP/DH 公钥校验缺失、协议 proof 可预测 | 发送特殊公钥后 shared secret 是否固定为 0 或小集合 | [auth-edge-cases-and-protocol-bypasses.md](auth-edge-cases-and-protocol-bypasses.md) |
| 表达式输出再被无 padding RSA 加密 | 是否能设计输出短明文、模板相关明文或可 Coppersmith 的已知前缀 | [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md)、[lattice-and-lwe.md](lattice-and-lwe.md) |
| Rabin/RSA LSB/noisy oracle | 每轮查询是否稳定给出一位、区间或带噪反馈 | [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md)、[number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md) |
| Hash state reversal、cycle detection、sponge MITM | 状态转移是否可逆、周期可测或可 meet-in-the-middle | 本页 raw |

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [ACTF2026-ohmycaptcha-wp](../raw/crypto/ACTF2026-ohmycaptcha-wp.md) | CAPTCHA 只是构造表达式的前置约束；核心 oracle 是“表达式输出被 `e=5` RSA 加密”，可组合小明文、Coppersmith 和 Franklin-Reiter。 |
| [RCTF2025-suanhash-wp](../raw/crypto/RCTF2025-suanhash-wp.md) | Sponge-like digest 暴露 rate/capacity 差分；两条相邻消息恢复内部差分后，下一块用互补差分直接碰撞。 |

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
- [ACTF2026-ohmycaptcha-wp](../raw/crypto/ACTF2026-ohmycaptcha-wp.md)
- [RCTF2025-suanhash-wp](../raw/crypto/RCTF2025-suanhash-wp.md)
