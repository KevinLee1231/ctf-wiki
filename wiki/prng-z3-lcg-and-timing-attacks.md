---
type: family
tags: [crypto, family, prng, z3, lcg, timing, constraint-solving]
skills: [ctf-crypto]
raw:
  - ../raw/crypto/prng-z3-lcg-and-timing-attacks.md
updated: 2026-06-12
---

# PRNG Z3, LCG and Timing Attacks

## 适用场景

PRNG 证据已经明确，但直接状态恢复不够：输出被截断、打乱、折叠、取模、混入业务反馈，或解题依赖 Z3 求解时间、网络时间、格式串写入等跨方向信号。

本页不并入 [mt-lcg-and-seed-recovery.md](mt-lcg-and-seed-recovery.md)，因为这里的首轮问题不是“是哪种 PRNG”，而是“随机状态如何通过约束、计时或外部 primitive 补齐”。

## 识别信号

- 只有 partial output、bit parity、subset sum、modulo 结果、ASCII 低位或经过 shuffle 的输出。
- 题目给出可交互 oracle，反馈不是明文随机数，而是超时、成功/失败、格式串可写、UUID 异常或协议状态。
- 需要把 PRNG 变量和其它约束同解，例如 DSA nonce、LFSR parity、Java LCG 低位、Rule 86 cellular automaton。
- 单纯 `randcrack`、untemper 或枚举 seed 无法解释全部样本。

## 最小证据

- 可复现的约束表达：输出位、取模值、比较条件、时间阈值或可控写入点。
- 变量规模可估计，能判断适合 Z3、MITM、线性代数、格或直接枚举。
- 至少一个用于筛选候选状态的外部校验点，例如签名验证、下一轮随机数、服务响应时间或 uuid/token 复算。

## 解法骨架

1. 把每个观察值写成状态变量上的约束，不急着选 solver。
2. 先做线性化检查：GF(2)、LCG 逆推、差分求模数能解决时，不要使用重型 SMT。
3. 变量较少或含非线性条件时再上 Z3；如果约束来自 timing，先测量阈值稳定性。
4. 状态恢复后回到原领域：签名 nonce 进 ECC/DSA，格式串进 Pwn，网络时间进 Forensics/Web。

## 关键变体

| 变体 | 触发证据 | 处理路线 |
|---|---|---|
| MT partial / subset sum | 只有部分 MT 输出或输出被集合/子集约束包住。 | 先写出 temper 后 bit 约束，能线性传播则线性求解，否则用 SMT/回溯补状态。 |
| Rule 86 / cellular automaton | 状态按局部规则演化，输出为末态或部分 bit。 | 建 bit-vector 或布尔约束，分块求解再拼接。 |
| Java LCG partial modulo | 输出经过 `% n` 或只泄露部分低位/高位。 | 逆推乘法、MITM 或剪枝枚举；截断严重时再考虑格。 |
| LFSR bit-fold parity | 输出是 ASCII parity、bit 折叠或过滤函数。 | 先尝试 GF(2) 线性系统；非线性过滤转 [rc4-lfsr-and-keystream-reuse.md](rc4-lfsr-and-keystream-reuse.md)。 |
| Z3 solve-time oracle | 服务响应时间泄露约束求解难度或分支。 | 固定网络抖动，重复采样取稳定阈值，再把 timing 转成离散条件。 |
| randcrack-fed DSA | PRNG 可预测签名 nonce 或 nonce 片段。 | PRNG 部分恢复后转 [ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md)。 |
| 格式串影响 seed/状态 | Pwn primitive 可改写全局随机状态、赌注或时间偏移。 | 先用 [format-string.md](format-string.md) 获得写原语，再回到 PRNG 预测。 |
| NTP/UUID/time poisoning | 网络时间或 UUID 结构泄露随机状态。 | 先确认流量和时间源，再转 [network-covert-auth-and-reassembly.md](network-covert-auth-and-reassembly.md) 补证据。 |

## 常见陷阱

- 约束过早 SMT 化，忽略 LCG 逆元、GF(2) 线性系统或直接枚举窗口。
- timing oracle 没有重复测量，网络抖动被误当成 bit。
- 只恢复 PRNG 状态，没有把状态继续带入签名、格式串或协议链。
- 把 LFSR/RC4 类 keystream 问题混进普通 seed recovery，导致模型错误。

## 关联技巧

- [mt-lcg-and-seed-recovery.md](mt-lcg-and-seed-recovery.md)
- [rc4-lfsr-and-keystream-reuse.md](rc4-lfsr-and-keystream-reuse.md)
- [lattice-and-lwe.md](lattice-and-lwe.md)
- [ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md)
- [format-string.md](format-string.md)
- [network-covert-auth-and-reassembly.md](network-covert-auth-and-reassembly.md)
- [crypto-tooling.md](crypto-tooling.md)

## 原始资料

- [prng-z3-lcg-and-timing-attacks.md](../raw/crypto/prng-z3-lcg-and-timing-attacks.md)
