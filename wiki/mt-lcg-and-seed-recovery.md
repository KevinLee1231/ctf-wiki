---
type: family
tags: [crypto, web, family, prng, mt19937, lcg, seed-recovery]
skills: [ctf-crypto, ctf-web]
raw:
  - ../raw/crypto/mt-lcg-and-seed-recovery.md
  - ../raw/web/WMCTF2025-guess-wp.md
updated: 2026-07-06
---

# MT, LCG and Seed Recovery

## 作用边界

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

## 分流流程

1. 先把 raw 输出统一成 PRNG 原始输出或约束：整数、bit、float mantissa、模后余数、被 XOR 的 keystream。
2. 判断能否直接状态恢复；能直接恢复时不要先上 Z3/格。
3. 如果输出不足、截断、打乱或带业务反馈，再转入 [prng-z3-lcg-and-timing-attacks.md](prng-z3-lcg-and-timing-attacks.md)。
4. 用恢复出的 seed/state 预测最短下一步，并回到原题业务链做正向验证。

## PRNG 路线分流

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

## 合并与拆分结论

- 保留为 family：MT19937、C `rand()`、LCG、V8 `Math.random()`、时间种子和 weak nonce 的实现不同，但首轮都要判断输出是否足够直接恢复 seed/state。
- 不与 [prng-z3-lcg-and-timing-attacks.md](prng-z3-lcg-and-timing-attacks.md) 合并：本页处理“可直接或近直接状态恢复”的路线，后者处理 partial output、timing、SMT、格式串写状态等需要外部约束补齐的路线。
- 不拆成 MT/LCG/V8 小页：当前 raw 更适合作为 PRNG 二级分流；若某一类积累多个独立 raw，再拆具体 technique。

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
| [D3CTF2022-d3bug-wp](../raw/crypto/D3CTF2022-d3bug-wp.md) | LFSR 输出直接泄露移出位，剩余低位可转成 GF(2) 线性方程或 SAT/Z3。 |
| [D3CTF2022-d3qcg-wp](../raw/crypto/D3CTF2022-d3qcg-wp.md) | 二次同余生成器只泄露高位，先把未知低位建成二元小根再恢复 PRNG 状态。 |
| [D3CTF2025-d3guess-wp](../raw/crypto/D3CTF2025-d3guess-wp.md) | 带噪声猜数反馈要用概率二分收集输出，再恢复 MT19937 状态预测后续随机数。 |
| [LilacCTF2026-noisy-forest-wp](../raw/crypto/LilacCTF2026-noisy-forest-wp.md) | Python MT bitstream 只以“中文字符是否 +9997”泄露；先用中文文本冗余恢复前缀，再从明密文差异克隆 MT19937 状态。 |
| [NCTF2026-rng-game-wp](../raw/crypto/NCTF2026-rng-game-wp.md) | 服务给出 Python `random` 大整数 seed，目标是构造另一 seed；按 32-bit 分块利用 CPython seed 扩展阶段碰撞。 |
| [RCTF2025-yet-another-mt-game-wp](../raw/crypto/RCTF2025-yet-another-mt-game-wp.md) | Sage `random_matrix(Zmod(mod))` 小模数路径泄露 GMP MT 31-bit 输出；用 GF(2) 线性系统恢复状态，再反推 GMP seed。 |
| [RCTF2025-yet-another-shuffled-mt-game-wp](../raw/crypto/RCTF2025-yet-another-shuffled-mt-game-wp.md) | GMP MT 输出被 Python `random.shuffle` 置乱；先从少量输出恢复 128-bit Python seed 逆置乱，再复用 GMP MT 状态恢复。 |
| [SUCTF2026-PrngWP](../raw/crypto/SUCTF2026-PrngWP.md) | 256-bit LCG 输出被“高低半 XOR + 按高位 ror”非线性包装；先恢复 rotation sequence/低半候选，再用 LCG 关系约束原 seed。 |
| [VNCTF2026-hd-is-what-wp](../raw/crypto/VNCTF2026-hd-is-what-wp.md) | SIDH/SIKE 公钥向量先被公开 seed 的 LCG 矩阵线性混淆；恢复标准公钥后再用 Castryck-Decru attack 求共享 j。 |
| [VNCTF2026-numberguesser-wp](../raw/crypto/VNCTF2026-numberguesser-wp.md) | 只有 10 次 hint 查询，但 Python `random.seed(os.urandom(8))` 可逆；选相隔 227 的输出 untemper/twist 反推 64-bit seed。 |
| [VNCTF2026-schnorr-wp](../raw/crypto/VNCTF2026-schnorr-wp.md) | Schnorr 服务固定 seed 导致首轮承诺 `B` 跨连接重复；对同一 `B` 给两个 challenge，用 special soundness 相减提 witness。 |

## 原始资料

- [mt-lcg-and-seed-recovery.md](../raw/crypto/mt-lcg-and-seed-recovery.md)
- [WMCTF2025-guess-wp](../raw/web/WMCTF2025-guess-wp.md)
