---
type: technique
tags: [reverse, crypto, vmp, smc, client-server, rc4, maze, dynamic-debugging]
skills: [ctf-reverse, ctf-crypto]
raw:
  - ../raw/reverse/Spirit2026-5-link-start-wp.md
updated: 2026-05-21
---

# VMP Client Server SMC RC4 Recovery

## 适用场景

题目同时给出 client / server，两端都被 VMProtect 或类似壳保护。交互协议先校验一段路径、迷宫或握手输入，再用该输入解密自修改代码，最后在解密出的函数里校验 flag。核心不是彻底还原壳，而是找到能稳定进入真实校验函数的最短路径。

## 识别信号

- client 和 server 均加壳，脱壳后仍有混淆但主逻辑可读。
- 第一段输入用于迷宫、握手或路径校验；第二段输入才是 flag。
- check 函数同时依赖两段输入，第一段还参与 SMC 解密。
- 解密后的内存出现正常 x64 prologue，函数中可见 RC4 KSA/PRGA 结构。

## 最小证据

- 已确认 client/server 双端至少一端被 VMP/壳/SMC 保护，且静态反编译直接读不到核心协议。
- 能抓到两端通信样本，或定位到加解密前后的 buffer。
- 至少恢复出一个稳定协议字段：长度、opcode、nonce、RC4 key、明密文对应关系或校验结果。

## 解法骨架

1. 先判断保护层：壳、SMC、反调试、虚拟化、动态解密字符串。
2. 不急着完整脱壳；优先动态断在 send/recv、加密函数、内存拷贝和比较点。
3. 抓 client/server 明密文对，确认是否为 RC4 或同类流密码：重复 key、KSA/PRGA 常量、逐字节 XOR。
4. 如果 key 在握手中派生，记录 nonce、时间戳、固定盐和双方字段。
5. 写脚本复现协议，先离线解出 flag，再决定是否需要联机 replay。

## 关键变体

| 变体 | 复用重点 |
|---|---|
| VMP 双端 | 不必完整理解所有 VM handler，先找 I/O 边界和明密文边界。 |
| SMC 解密窗口 | 代码或字符串只在运行时短暂出现，需动态 dump。 |
| RC4 协议 | KSA 256 次置换和 PRGA 输出字节是强识别信号。 |
| Client/server 交叉验证 | 一端看不懂时，另一端可能暴露相同 key 或协议字段。 |

## 常见陷阱

- 把目标设成完整反虚拟化，忽略 I/O 边界可直接恢复协议。
- 只分析 client，不利用 server 作为 oracle 或对照实现。
- 动态 dump 没记录输入条件，导致结果不可复现。
- 把 RC4 明密文 XOR 当最终答案，忘记 key 派生和消息 framing。

## 关联技巧

- [vm-obfuscation-transform-family.md](vm-obfuscation-transform-family.md)
- [packers-deobfuscation-and-debug-automation.md](packers-deobfuscation-and-debug-automation.md)
- [runtime-patching-oracles-and-tracing.md](runtime-patching-oracles-and-tracing.md)
- [rc4-lfsr-and-keystream-reuse.md](rc4-lfsr-and-keystream-reuse.md)
- [game-state-websocket-and-wasm.md](game-state-websocket-and-wasm.md)

## 原始资料

- [Spirit2026-5-link-start-wp.md](../raw/reverse/Spirit2026-5-link-start-wp.md)
