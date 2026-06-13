---
type: family
tags: [crypto, family, zkp, secret-sharing, garbled-circuit, pairing, proof-system]
skills: [ctf-crypto]
raw:
  - ../raw/crypto/zkp-secret-sharing-and-proof-systems.md
updated: 2026-06-12
---

# ZKP, Secret Sharing and Proof Systems

## 作用边界

本页是证明系统、秘密分享和约束电路 family，覆盖 ZKP、Graph 3-coloring、Z3/SMT、garbled circuits、Shamir secret sharing、Groth16、DV-SNARG、KZG/pairing oracle、MAYO/FROST 类签名结构和 crypto-protected endpoint race。

它不是“用 Z3 解题”的 technique。首轮要判断证据属于约束求解、电路元数据泄露、可信设置错误、nullifier 未约束、pairing oracle、秘密分享系数复用，还是签名协议/阈值协议中的结构错误。

## 共同识别信号

- 题目给出 proof、verifier、circuit、witness、commitment、share、pairing 点、garbled table、Z3 约束或可查询 verifier oracle。
- 验证通过与否比密文本身更重要，攻击目标可能是 forge proof、recover witness、recover key、重放 proof 或重构 secret。
- 随机性、trusted setup、nullifier、metadata、共享多项式或阈值签名 nonce/commitment 有复用或缺约束。

## 最小证据

- 明确验证方检查了什么、没检查什么：public input、witness、nullifier、commitment、pairing equation、share index。
- 能写出一条最小约束或等式，解释为什么 proof/share/circuit 可被伪造或恢复。
- 对 oracle 类题，记录查询预算、响应类型、重放条件和安全失败状态。
- 对 Z3/SMT，先抽取真实约束，再做 printable/范围/位宽建模。

## 首轮路由

| 证据形态 | 首轮判断 | 下一跳 |
|---|---|---|
| 普通 bitvector/字符串约束、BPF/seccomp 规则 | 先把约束转成可执行模型，确认位宽和端序 | [oracles-recurrences-captcha-polyglots.md](oracles-recurrences-captcha-polyglots.md) |
| Garbled circuit、Free XOR、metadata 泄露 | 先找全局 delta、wire label 关系和 AES key 派生边界 | [block-mode-misuse-family.md](block-mode-misuse-family.md) |
| Shamir share 系数确定或复用 | 先写多项式关系和 share index，再做插值、差分或根查找 | [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md) |
| Groth16 trusted setup、nullifier、proof replay | 先展开 verifier pairing equation，找未约束变量或可消项 | [crypto-tooling.md](crypto-tooling.md) |
| DV-SNARG/KZG/pairing oracle | 先判断 oracle 暴露的是大小、符号、排列还是 pairing 等式 | [homomorphic-and-exotic-algebra.md](homomorphic-and-exotic-algebra.md), [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md) |
| MAYO/FROST/阈值签名结构错误 | 先确认 nonce、commitment、share 或矩阵结构是否复用/可扰动 | [lattice-and-lwe.md](lattice-and-lwe.md), [ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md) |
| crypto-protected endpoint 竞争 | 先固定并发窗口和签名修改方式，再转 Web/Pwn race | [race-condition-and-concurrency-exploits.md](race-condition-and-concurrency-exploits.md) |

## 合并与拆分结论

本页应保留为 family。ZKP、garbled circuit、Shamir、KZG 和阈值签名的攻击模型不同，但都属于“验证关系或共享关系缺约束”的二级路由。当前 raw 还不足以拆成多个稳定 technique。

## 常见陷阱

- 只把 proof 当黑盒，没展开 verifier 等式。
- Z3 模型没设置位宽，导致解和程序不一致。
- Shamir 题默认每个字符独立多项式，忽略系数复用。
- Pairing oracle 中把点顺序、群 G1/G2 或 `psi` 映射搞反。
- Race 题只测单请求，没复现并发窗口。

## 关联技巧

- [crypto-parameter-triage-family.md](crypto-parameter-triage-family.md)
- [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md)
- [homomorphic-and-exotic-algebra.md](homomorphic-and-exotic-algebra.md)
- [lattice-and-lwe.md](lattice-and-lwe.md)
- [ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md)
- [oracles-recurrences-captcha-polyglots.md](oracles-recurrences-captcha-polyglots.md)
- [crypto-tooling.md](crypto-tooling.md)

## 原始资料

- [zkp-secret-sharing-and-proof-systems.md](../raw/crypto/zkp-secret-sharing-and-proof-systems.md)
