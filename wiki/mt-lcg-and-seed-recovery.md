---
type: family
tags: [crypto, family, prng, mt19937, lcg, seed-recovery]
skills: [ctf-crypto]
raw:
  - ../raw/crypto/mt-lcg-and-seed-recovery.md
  - ../raw/web/WMCTF2025-guess-wp.md
updated: 2026-06-12
---

# MT, LCG and Seed Recovery

## 适用场景

题目核心是从弱随机数恢复 seed、内部状态或下一轮输出，常见于抽奖、guess game、token/key 生成、nonce 生成、密码学包装层和 Web 鉴权流程。

本页是 PRNG 状态恢复的二级 family 页，不是单一 technique。进入本页前应先确认主要障碍确实是“随机数可预测”，而不是分组模式、RSA 参数或普通编码。

## 识别信号

- 能看到连续或近连续的随机输出，例如整数、浮点数、token 字段、验证码、抽奖结果或签名 nonce。
- 源码出现 `random`、`mt19937`、`srand/rand`、LCG、middle-square、logistic map、`Math.random()` 或自定义递推。
- seed 来自时间、flag 字节、用户名、短密码、可枚举参数或服务启动时间。
- PRNG 输出后还接 Web eval、签名、加密、哈希或业务校验，恢复状态只是漏洞链的一段。

## 最小证据

- 生成器类型或足够缩小的候选实现，例如 Python MT19937、C libc `rand()`、Java LCG、V8 XorShift128+。
- 输出位宽、输出数量和输出顺序；如果输出被截断或打乱，要先记录截断位置和可用约束。
- seed 来源与枚举窗口，例如秒级时间戳、固定用户名、短 flag 前缀或可复现服务启动时间。
- 一个正向校验点：预测下一次输出、复算 token、通过签名验证或还原明文。

## 解法骨架

1. 先把 raw 输出统一成 PRNG 原始输出或约束：整数、bit、float mantissa、模后余数、被 XOR 的 keystream。
2. 判断能否直接状态恢复；能直接恢复时不要先上 Z3/格。
3. 如果输出不足、截断、打乱或带业务反馈，再转入 [prng-z3-lcg-and-timing-attacks.md](prng-z3-lcg-and-timing-attacks.md)。
4. 用恢复出的 seed/state 预测最短下一步，并回到原题业务链做正向验证。

## 关键变体

| 变体 | 触发证据 | 处理路线 |
|---|---|---|
| MT19937 32 bit 输出 | 624 个连续 `getrandbits(32)` 或等价 32 bit 输出。 | untemper 恢复 state，预测下一轮；若输出来自 Web 鉴权，恢复后转回 Web 链。 |
| Python `random.random()` 浮点 | 多个 53 bit 浮点值或可由浮点反推的 mantissa。 | 建 GF(2) 线性关系或使用 float-state solver，还原 MT state。 |
| 时间种子 | 输出可复算，seed 落在秒级/毫秒级短窗口。 | 枚举时间窗口，先固定时区和服务启动点，再用首个输出筛掉候选。 |
| C `srand/rand` | 服务端使用 libc `rand()`，输出和平台相关。 | 用相同 libc 或 `ctypes` 复现，注意 RAND_MAX、取模偏差和多进程状态。 |
| LCG 参数或状态恢复 | 输出满足线性递推，可能只泄露高位或模后值。 | 参数未知时用差分求模数；截断输出转 [lattice-and-lwe.md](lattice-and-lwe.md) 或约束求解。 |
| V8 `Math.random()` | Node/Chrome 服务用 `Math.random()` 生成 token。 | 按 V8 XorShift128+ 和 cache 顺序反推 state，注意输出缓存方向。 |
| chaotic / middle-square / flag-derived RNG | seed 空间小、状态退化或由 flag 字节确定。 | 先找不变量和退化周期，再 bounded brute force 或 hill climbing。 |
| PRNG 喂给 DSA/ECDSA nonce | 签名随机数由弱 PRNG 产生。 | 先恢复 k 或 k 的约束，再转 [ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md)。 |

## 常见陷阱

- 忽略 PRNG 实现版本：Python、glibc、musl、Java、V8 的状态和输出转换不同。
- 把取模后的结果当完整输出，导致状态恢复条件不成立。
- 收集样本时跨进程、跨用户或跨重启，输出序列不连续。
- 预测成功后没有回到业务层，漏掉 Web eval、签名校验或后续解密步骤。

## 关联技巧

- [prng-z3-lcg-and-timing-attacks.md](prng-z3-lcg-and-timing-attacks.md)
- [rc4-lfsr-and-keystream-reuse.md](rc4-lfsr-and-keystream-reuse.md)
- [lattice-and-lwe.md](lattice-and-lwe.md)
- [ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md)
- [web-first-pass-triage-and-chain-patterns.md](web-first-pass-triage-and-chain-patterns.md)
- [crypto-tooling.md](crypto-tooling.md)

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [WMCTF2025-guess-wp](../raw/web/WMCTF2025-guess-wp.md) | Web 注册接口连续返回 32 bit MT19937 输出，收集 624 个 `getrandbits(32)` 后预测鉴权 key；后半段需要回到 Web eval 逃逸。 |

## 原始资料

- [mt-lcg-and-seed-recovery.md](../raw/crypto/mt-lcg-and-seed-recovery.md)
- [WMCTF2025-guess-wp](../raw/web/WMCTF2025-guess-wp.md)
