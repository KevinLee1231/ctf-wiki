---
type: family
tags: [crypto, family, rsa, coppersmith, oracle, signature]
skills: [ctf-crypto]
raw:
  - ../raw/crypto/rsa-specialized-structures-and-oracles.md
  - ../raw/crypto/0xGame2020-week1-Fibonacci-wp.md
  - ../raw/crypto/LilacCTF2026-myrsa-wp.md
  - ../raw/crypto/NCTF2026-ez-rsa-wp.md
  - ../raw/crypto/NCTF2026-hard-rsa-wp.md
  - ../raw/crypto/SUCTF2026-RSAWP.md
  - ../raw/crypto/VNCTF2026-math-rsa-wp.md
updated: 2026-07-27
---

# RSA Specialized Structures and Oracles

## 作用边界

本页是 RSA 长尾结构和 oracle family。它承接那些已经超出基础 RSA 排查的题：特殊素数关系、`gcd(e, phi)` 非 1、CRT 参数泄露、同态签名、fault、ROCA、padding oracle、LSB oracle、可构造模数、长度/截断 bug 和 Coppersmith 小根模型。

基础小 e、共模、Hastad、Wiener、Fermat、多素数等快速排查先看 [rsa-attacks.md](rsa-attacks.md)。

## 识别信号

- RSA 参数带有额外结构：prime 线性/多项式关系、递推生成、已知 bit、CRT 泄露、fault、ROCA、可构造 modulus 或实现层截断。
- 题目给出 hint、oracle、签名接口、验签逻辑、padding 错误、LSB/区间反馈或可控消息变换。
- 常规 `rsa-attacks.md` 快速排查失败，但参数规模、泄露位数或响应差异明显暗示可建模。
- 需要 Coppersmith、Boneh-Durfee、CM/曲线 trace、CRT 枚举、oracle 区间收缩或实现 bug 才能推进。

## 最小证据

- 写清未知量和已知量边界：泄露哪些位、未知范围多大、小根变量个数、oracle 每次给什么反馈。
- 对结构化 prime，必须能把题面 hint 转成等式或不等式，并估计是否满足小根/近似分解条件。
- 对 oracle，至少证明响应差异稳定、可重复，且能单调收缩明文区间或候选集合。
- 对签名/实现 bug，要保留验签/解析顺序、padding 检查位置、截断/NULL/last-byte 行为和可控消息格式。

## 变体路由

| 信号 | 先确认 | 下一跳 |
|---|---|---|
| prime 有线性关系、已知高/低/中间位、base representation | 未知量界是否满足 Coppersmith，小根变量是一元还是多元 | [lattice-and-lwe.md](lattice-and-lwe.md)、[number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md) |
| prime 多项式结构可转 CM trace 或曲线阶关系 | 是否能把素数写成 `4p=t^2+Dy^2`，以及 oracle 是否给出曲线点 | [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md) |
| prime 由递推序列、周期和或 `next_prime(S^k)` 生成 | 先恢复生成 `S` 的周期/闭式，再用 `p*q == N` 验证分解 | [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md) |
| 小私钥或 RSA-like `phi_n(N)` | 连分数对象、误差界和后续 high-bits-known factoring 条件 | [lattice-and-lwe.md](lattice-and-lwe.md) |
| `gcd(e, phi(n)) > 1`、多重根、prime power | 每个模 prime 下有多少根，是否能 CRT 枚举并用格式筛选 | 本页 raw |
| 泄露 `dp/dq/qinv`、CRT fault、bit flip | CRT 参数能否反推出 p/q，或故障签名和正确签名的 gcd 是否暴露因子 | 本页 raw |
| 签名同态、message factoring、`e=1`、可构造 modulus | 验证逻辑是否只检查 textbook RSA 关系或 PKCS 前缀 | [hash-protocol-and-oracle-attacks.md](hash-protocol-and-oracle-attacks.md) |
| decryption oracle、LSB oracle、Manger/Bleichenbacher | 响应差异是否可稳定压缩明文区间 | [hash-protocol-and-oracle-attacks.md](hash-protocol-and-oracle-attacks.md) |
| Batch GCD、shared prime、ROCA | 是否有多公钥集合、模数指纹或可批量 gcd 的输入 | [crypto-tooling.md](crypto-tooling.md) |
| strlen/NULL/last-byte overwrite、协议包装 bug | RSA 漏洞是否来自实现层截断或解析差异 | [json-duplicate-key-hmac-parser-differential.md](json-duplicate-key-hmac-parser-differential.md) |

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [0xGame2020-week1-Fibonacci-wp](../raw/crypto/0xGame2020-week1-Fibonacci-wp.md) | `p = next_prime(S**16)` 且 `S` 来自 Fibonacci 模小整数的长和；先用 Pisano period 快速恢复 `S`，再验证 `p*q == N` 并常规解密。 |
| [LilacCTF2026-myrsa-wp](../raw/crypto/LilacCTF2026-myrsa-wp.md) | 三素数 RSA 中 `p=pp^2+3pp+3` 可整理成 CM 曲线 trace 关系；平方根 oracle 给出一个点后，可用 ECM 型点乘和 `gcd` 分解 `n`。 |
| [NCTF2026-ez-rsa-wp](../raw/crypto/NCTF2026-ez-rsa-wp.md) | hint 从两端约束 `p/q` bit；DFS 时同时检查乘积上下界和低位乘积模约束。 |
| [NCTF2026-hard-rsa-wp](../raw/crypto/NCTF2026-hard-rsa-wp.md) | RSA-like `phi_6(N)` 需要 generalized Wiener：连分数恢复小参数，再把 `p_hat` 近似转成 Coppersmith 或近邻分解。 |
| [SUCTF2026-RSAWP](../raw/crypto/SUCTF2026-RSAWP.md) | 小 `d` 与 `p+q` 高位泄露组合成 Boneh-Durfee/Coppersmith 二元小根；格构造要按 `k,x` 的绑定关系调整 shift。 |
| [VNCTF2026-math-rsa-wp](../raw/crypto/VNCTF2026-math-rsa-wp.md) | 大整数 `k` 不是分解线索，而是可因式分解的 `phi` 恒等式；枚举短参数即可恢复 `phi` 直接解密。 |
| [D3CTF2019-bivariate-wp](../raw/crypto/D3CTF2019-bivariate-wp.md) | RSA 素数中间位泄露可写成二元小根，先估计未知高低位界并构造 Coppersmith 格。 |
| [D3CTF2019-common-wp](../raw/crypto/D3CTF2019-common-wp.md) | 共模 RSA 小私钥指数变体，先把 Wiener/Guo 方程转成低维格再恢复 phi。 |
| [D3CTF2022-d3factor-wp](../raw/crypto/D3CTF2022-d3factor-wp.md) | Prime Power RSA 的指数差满足小根方程，先在 p 的高次幂模意义下构造 Coppersmith。 |
| [D3CTF2023-d3sys-wp](../raw/crypto/D3CTF2023-d3sys-wp.md) | CTR-SM4 token 可篡改后进入 CRT-RSA partial key exposure，重点估泄露位和 Coppersmith bound。 |
| [D3CTF2025-d3sys-p2-2-0-wp](../raw/crypto/D3CTF2025-d3sys-p2-2-0-wp.md) | RSA/CRT 因子有已知线性近似且修正项很小，优先建 partial factoring 小根模型。 |
| [HGAME2026-babyrsa-wp](../raw/crypto/HGAME2026-babyrsa-wp.md) | `p,q` 公开但模数小于完整 flag，密文只约束 `m mod n`；结合前后缀、长度和字符集用格恢复原明文字节。 |
| [HGAME2026-ezrsa-wp](../raw/crypto/HGAME2026-ezrsa-wp.md) | 指数 bit-flip 加密 oracle 可恢复 `n` 和 50-bit `e`，拿到 flag 密文后再用解密 oracle 做低字节推进。 |
| [ACTF2026-zjuam-just-uses-awful-math-wp](../raw/crypto/ACTF2026-zjuam-just-uses-awful-math-wp.md) | HTTP 抓包里暴露弱 RSA 公钥和密文；载体是 HTTP，但决定性主障碍是 Crypto/RSA 参数分解。 |
| [0xGame2025-week3-where-is-my-public-key-wp](../raw/crypto/0xGame2025-week3-where-is-my-public-key-wp.md) | 本题并不是缺少整个 RSA 公钥，而是只缺少指数 $e$；反而直接泄露了本应保密的 $p,q$；选择明文 oracle 给出了 $m^e\bmod n$，可把未知指数变成离散对数问题。 |
| [UMDCTF2023-reduce-thyself-wp](../raw/crypto/UMDCTF2023-reduce-thyself-wp.md) | 用 RSA 乘法同态对目标密文做随机盲化，使其满足 oracle 的“密文整数为素数”过滤条件，再去盲化明文；RSA 解密 oracle 只按密文表示做白名单或黑名单过滤，却没有使用 OAEP 等非延展编码时，应测试乘法自归约。 |
| [UMDCTF2025-square-up-wp](../raw/crypto/UMDCTF2025-square-up-wp.md) | 利用“部分正确”的错误 CRT 结果，通过 $\gcd(r^2-c,N)$ 提取仍然成立的模因子；RSA/Rabin 程序公开了解密失败输出、故障签名或只有一个 CRT 分支被污染的结果。 |
| [UMDCTF2026-no-brainrot-allowed-wp](../raw/crypto/UMDCTF2026-no-brainrot-allowed-wp.md) | 只泄露一个前缀判断并不代表 RSA 解密接口安全。教科书 RSA 的乘法同态把这一位信息放大成可重复的区间查询，攻击者能够逐轮缩小明文范围。实际系统应使用经过证明的填充方案，并彻底移除可区分的解密错误。 |
| [WMCTF2023-badprime-wp](../raw/crypto/WMCTF2023-badprime-wp.md) | 利用 RSA 素数生成中的 ROCA/CVE-2017-15361 类结构，使 `p` 落入可预测同余类，再用 Coppersmith 小根恢复因子；素数生成出现 `pow(e, random, M) + k*M`、服务端泄露 `p mod leak`、且 `leak` 必须整除固定 `M` 时，应考虑构造 `p mod v`。 |
| [WMCTF2023-welcome-signer1-wp](../raw/crypto/WMCTF2023-welcome-signer1-wp.md) | 对 RSA 模幂过程中的公钥模数做故障注入，通过正确签名和故障签名的代数关系逐位恢复私钥指数；签名服务允许“改变 N 的一个字节”并指定干扰位置 `j`，同时返回同一固定消息的签名时，应考虑 RSA public key perturbation fault attack。 |
| [WMCTF2023-welcome-signer2-wp](../raw/crypto/WMCTF2023-welcome-signer2-wp.md) | 针对 Right-to-Left RSA 模幂的 public modulus fault attack，根据故障发生位置从高位恢复私钥指数；模幂实现维护 `A` 和 `B` 两个变量，且故障在接近指数末尾时切换到 $\hat N$，应分析 `B` 链对后续乘法的影响。 |
| [WMCTF2024-facrt-wp](../raw/crypto/WMCTF2024-facrt-wp.md) | 故障不是直接给一组错误签名求 gcd，而是多组“小余数”签名构成 Partial ACD；清零高 32 位让 $s_q^*$ 足够小，满足格攻击条件。 |

## Technique 下一跳

| 首轮判断 | 具体 technique |
|---|---|
| 特殊 prime、CRT 参数或部分私钥关系可转为因子约束 | [rsa-factor-relation-and-partial-key-recovery.md](rsa-factor-relation-and-partial-key-recovery.md) |
| OAEP/PKCS#1/区间/timing 反馈可自适应收缩明文 | [rsa-padding-and-interval-oracle-attacks.md](rsa-padding-and-interval-oracle-attacks.md) |
| 已知位、短差值或有界未知量满足模多项式 | [lattice-small-root-and-partial-leakage.md](lattice-small-root-and-partial-leakage.md) |

## 合并与拆分结论

- 保留为 family：raw 覆盖大量互不相同的 RSA 长尾结构，页面价值是判断建模路线。
- 不与 `rsa-attacks.md` 合并：基础页用于快速排查，specialized 页用于需要建模、oracle 或实现 bug 的题。
- 已把结构化因子/部分密钥、RSA 区间 oracle 和格小根三条高复用路线接到具体 technique；其余低频结构仍保留为 case routing。

## 常见误判

- 把所有 RSA 都先丢给通用工具，忽略题目给的结构关系。
- Coppersmith 未先估未知量界，导致格维度和 bound 乱试。
- Oracle 只看一次响应，没有证明响应差异和明文区间存在稳定关系。
- 签名题只关注 hash 算法，忽略 textbook RSA 乘法同态和验签前缀检查。

## 关联页面

- [crypto-parameter-triage-family.md](crypto-parameter-triage-family.md)
- [rsa-attacks.md](rsa-attacks.md)
- [lattice-and-lwe.md](lattice-and-lwe.md)
- [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md)
- [hash-protocol-and-oracle-attacks.md](hash-protocol-and-oracle-attacks.md)
- [crypto-tooling.md](crypto-tooling.md)

## 原始资料

- [rsa-specialized-structures-and-oracles.md](../raw/crypto/rsa-specialized-structures-and-oracles.md)
- [0xGame2020-week1-Fibonacci-wp](../raw/crypto/0xGame2020-week1-Fibonacci-wp.md)
- [LilacCTF2026-myrsa-wp](../raw/crypto/LilacCTF2026-myrsa-wp.md)
- [NCTF2026-ez-rsa-wp](../raw/crypto/NCTF2026-ez-rsa-wp.md)
- [NCTF2026-hard-rsa-wp](../raw/crypto/NCTF2026-hard-rsa-wp.md)
- [SUCTF2026-RSAWP](../raw/crypto/SUCTF2026-RSAWP.md)
- [VNCTF2026-math-rsa-wp](../raw/crypto/VNCTF2026-math-rsa-wp.md)
