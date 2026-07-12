---
type: family
tags: [crypto, family, ecc, dlp, signature, nonce]
skills: [ctf-crypto]
raw:
  - ../raw/crypto/ecc-dlp-and-signature-attacks.md
updated: 2026-07-06
---

# ECC DLP and Signature Attacks

## 作用边界

题目核心是曲线群、DLP、签名 nonce、small subgroup、invalid curve、异常曲线、同源/配对或 DSA/ECDSA/Schnorr 类签名关系。它是 crypto 首轮分流后的 ECC/签名 family 页，不是单一攻击技巧页。

## 识别信号

- 给出曲线参数、基点、点坐标、签名 `(r, s)`、公钥、oracle 或可控输入点。
- 曲线阶、子群阶、cofactor、判别式、j-invariant、异常阶或奇怪基点明显可疑。
- 多个签名存在相同 `r`、相近 nonce、低熵 nonce、hash collision 或 PRNG 生成的 nonce。
- 服务端没有检查点是否在曲线上、是否在正确子群中，或返回标量乘结果/验证结果差异。

## 最小证据

- 明确曲线定义、模数、基点阶和目标未知量：私钥、nonce、离散对数、共享密钥或明文。
- 至少一种可验证路线：群阶分解、签名方程、oracle 查询差异、点映射或 nonce 约束。
- 如果是签名问题，要保留消息哈希算法、签名格式和每个签名的 `(r, s, h)`。
- 如果是曲线结构问题，要先验证点合法性、阶、cofactor 和是否退化到更简单群。

## 分流流程

1. 先做参数体检：曲线是否标准、阶是否 smooth、基点是否正确、点是否在曲线上。
2. DLP 路线先看群阶分解和可降维结构；签名路线先写出 nonce 方程。
3. 有 oracle 时先构造最小查询，确认是 small subgroup、invalid curve、torsion side channel 还是普通验证差异。
4. 得到候选私钥/nonce 后，用签名验证、标量乘或共享密钥派生做正向复算。

## ECC / 签名路线分流

| 变体 | 触发证据 | 处理路线 |
|---|---|---|
| smooth order DLP | 群阶可分解为小因子或 clock group 结构明显。 | Pohlig-Hellman / BSGS 分块求解，再 CRT 合并。 |
| invalid / singular / anomalous curve | 点不在曲线上、判别式为 0、阶等于 p 或曲线退化。 | 验证曲线结构，映射到乘法/加法群或使用 Smart attack。 |
| small subgroup / torsion | cofactor 大，服务端接受低阶点或 Ed25519 torsion 差异。 | 查询低阶点泄露私钥模小因子，收集后 CRT。 |
| repeated ECDSA/DSA nonce | 多签名 `r` 相同或 k 可复用。 | 由两个签名恢复 k 和私钥，再验证公钥。 |
| partial / biased nonce | nonce 高位/低位泄露、短 nonce、PRNG nonce 或 timing 泄露。 | 写 HNP/EHNP 关系，转 [lattice-and-lwe.md](lattice-and-lwe.md) 或 PRNG 页补状态。 |
| hash-collision generated k | nonce 派生依赖 MD5/SHA 碰撞或弱 hash。 | 先转 [hash-protocol-and-oracle-attacks.md](hash-protocol-and-oracle-attacks.md) 固定碰撞关系，再回签名方程。 |
| shared prime / modulus factor | 多个曲线或签名参数共享素因子。 | GCD 先做参数恢复，再进入 DLP/签名路线。 |

## 合并与拆分结论

- 保留为 family：曲线参数体检、DLP、invalid/singular/anomalous curve、small subgroup 和签名 nonce 恢复的第一步判断不同，但都围绕曲线群或签名方程的结构缺陷。
- 不并入 [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md)：ECC/签名题需要先处理点合法性、群阶、cofactor、签名格式和 nonce 方程，和一般数论代数页的入口证据不同。
- 不拆成 DLP、invalid curve、signature nonce 三个 technique：当前页面承担 Crypto triage 的二级分流；具体 HNP/格、PRNG nonce 或 hash collision 会继续转到相邻页面。

## 常见陷阱

- 只看曲线名，不检查实际参数、基点阶和 cofactor。
- 把 Schnorr、DSA、ECDSA 的签名方程混用，导致 nonce 恢复公式错误。
- 低阶点 oracle 没有收集足够模因子，CRT 结果不唯一。
- partial nonce 已进入格问题时，仍尝试暴力完整 nonce。

## 关联技巧

- [crypto-parameter-triage-family.md](crypto-parameter-triage-family.md)
- [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md)
- [lattice-and-lwe.md](lattice-and-lwe.md)
- [mt-lcg-and-seed-recovery.md](mt-lcg-and-seed-recovery.md)
- [hash-protocol-and-oracle-attacks.md](hash-protocol-and-oracle-attacks.md)
- [crypto-tooling.md](crypto-tooling.md)

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [D3CTF2019-babyecc-wp](../raw/crypto/D3CTF2019-babyecc-wp.md) | 曲线参数和 ECC 群运算是核心，先确认曲线阶、基点和标量关系。 |
| [D3CTF2019-sign2win-wp](../raw/crypto/D3CTF2019-sign2win-wp.md) | DSA/ECDSA 签名或 nonce 关系异常，先恢复签名方程中的随机数/私钥约束。 |
| [D3CTF2021-easy-curve-wp](../raw/crypto/D3CTF2021-easy-curve-wp.md) | 曲线/DLP 参数是主约束，先确认群阶、基点、映射和可降维关系。 |
| [D3CTF2022-leak-dsa-wp](../raw/crypto/D3CTF2022-leak-dsa-wp.md) | DSA/ECDSA nonce 或签名泄露进入格恢复，先写出每个签名的线性近似关系。 |
| [HGAME2026-ezcurve-wp](../raw/crypto/HGAME2026-ezcurve-wp.md) | ECC 横坐标 oracle 每次减去 163-bit 噪声；成对查询 `t/-t` 消去部分曲线关系，再把小误差放进 LLL。 |
| [RCTF2025-repairing-wp](../raw/crypto/RCTF2025-repairing-wp.md) | Pairing/ElGamal 风格密文可重随机化：同时改 `C1,C2,C3` 保持 shared key 不变，服务端未去重时可解出原 flag key。 |
| [SUCTF2026-IsogenyWP](../raw/crypto/SUCTF2026-IsogenyWP.md) | CSIDH 群作用与 2-isogeny 交换，三条 2-isogenous 共享曲线高位泄露可建 CI-HNP 小根方程，用 Automated Coppersmith 恢复参数。 |
| [VNCTF2026-hd-is-what-wp](../raw/crypto/VNCTF2026-hd-is-what-wp.md) | SIDH/SIKE 公钥向量先被公开 seed 的 LCG 矩阵线性混淆；恢复标准公钥后再用 Castryck-Decru attack 求共享 j。 |
| [VNCTF2026-schnorr-wp](../raw/crypto/VNCTF2026-schnorr-wp.md) | Schnorr 服务固定 seed 导致首轮承诺 `B` 跨连接重复；对同一 `B` 给两个 challenge，用 special soundness 相减提 witness。 |
| [D3CTF2019-keygenme-wp](../raw/reverse/D3CTF2019-keygenme-wp.md) | Keygen 逻辑落到签名/曲线数学关系，先把校验式转成可求解的 DSA/ECDSA 约束。 |

## 原始资料

- [ecc-dlp-and-signature-attacks.md](../raw/crypto/ecc-dlp-and-signature-attacks.md)
