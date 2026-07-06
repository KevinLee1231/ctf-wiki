---
type: family
tags: [crypto, family, triage]
skills: [ctf-crypto]
updated: 2026-07-06
---

# Crypto Parameter and Oracle Triage

## 作用边界

密文、参数、签名、oracle、PRNG 输出或自定义数学协议同时出现，首轮还不能确定是 RSA、ECC、格、PRNG、分组模式还是哈希协议。

本页是 WP 摄入后的 pivot 页：它不按比赛或题目组织，而是把 raw WP 中重复出现的识别信号压缩成下一跳判断表。

## 识别信号

- 题面给出 n/e/c、曲线参数、签名、seed、oracle、PoW、验证码或自定义方程。
- 参数规模、可控查询或错误反馈能决定下一跳攻击族。
- 同一 WP 中可能同时出现数论和工程包装；先抽出数学关系再考虑工具。

## 最小证据

- 至少有一组可复算的参数、可重放的输入输出、可观察的错误/状态差异，或可定位的文件/交互边界。
- 能说明当前题更接近下表哪个下一跳技巧页，而不是只按比赛名或题名判断。
- 若多个变体同时出现，先选择能最小验证的信号；失败后再按相邻技巧 pivot。

## 分流流程

1. 列出 known / unknown / goal 和所有可复算方程。
2. 按参数形态进入 RSA、ECC、格、PRNG、分组模式、哈希协议或代数页。
3. 做一个最小验证：因数分解、小根、状态恢复、oracle 差异或明密文复算。
4. 若失败，回到参数规模和泄露边界，换相邻技巧页。

## 首轮参数分流

| 下一跳 | 触发信号 | WP 数 |
|---|---|---:|
| [block-mode-misuse-family.md](block-mode-misuse-family.md) | AES/分组模式、IV、padding、MAC 或加解密 oracle。 | 2 |
| [ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md) | 曲线/DLP/签名/同源/配对关系。 | 10 |
| [exotic-secret-sharing-rabin-and-polynomials.md](exotic-secret-sharing-rabin-and-polynomials.md) | secret sharing、Rabin、扰动多项式或可替换私钥结构。 | 1 |
| [hash-protocol-and-oracle-attacks.md](hash-protocol-and-oracle-attacks.md) | 哈希、PoW、协议校验或弱比较。 | 3 |
| [homomorphic-and-exotic-algebra.md](homomorphic-and-exotic-algebra.md) | 同态、配对或特殊代数结构。 | 6 |
| [lattice-and-lwe.md](lattice-and-lwe.md) | 格规约、小根、近似约束或高维线性关系。 | 21 |
| [mt-lcg-and-seed-recovery.md](mt-lcg-and-seed-recovery.md) | PRNG seed、状态恢复、MT/LCG/LFSR 输出泄露。 | 11 |
| [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md) | 数论、代数方程、枚举边界或自定义数学结构。 | 15 |
| [oracles-recurrences-captcha-polyglots.md](oracles-recurrences-captcha-polyglots.md) | 递推、验证码、表达式 oracle 或跨域约束系统。 | 3 |
| [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md) | RSA 模数、素数结构、小根、低指数或 oracle。 | 15 |
| [rc4-lfsr-and-keystream-reuse.md](rc4-lfsr-and-keystream-reuse.md) | 流密码、LFSR、RC4、XOR keystream 或 bit 级线性密钥流。 | 2 |
| [zkp-secret-sharing-and-proof-systems.md](zkp-secret-sharing-and-proof-systems.md) | proof、verifier、circuit、witness、commitment、share、nullifier、pairing oracle 或阈值协议缺约束。 | 1 |

## WP 案例沉淀

本节只抽取 raw WP 中可复用的识别信号和下一跳，不替代原始题解正文。

| Raw WP | 复用信号 | 下一跳 |
|---|---|---|
| [WMCTF2025-ishowsplit-wp](../raw/crypto/WMCTF2025-ishowsplit-wp.md) | `A*x+r*k=b mod p` 且隐藏数有不连续 bit 泄露，先把已知段合入 `xbar`，未知段走 EHNP LLL/Babai CVP。 | [lattice-and-lwe.md](lattice-and-lwe.md) |
| [WMCTF2025-lw3-wp](../raw/crypto/WMCTF2025-lw3-wp.md) | LWE error 不小但只来自少数固定随机值，先把 error 选择展开成 0/1 变量，再降维/精确 SVP。 | [lattice-and-lwe.md](lattice-and-lwe.md) |
| [WMCTF2025-lw5-wp](../raw/crypto/WMCTF2025-lw5-wp.md) | 可自选 LWE error 集且服务端阻止简单小化，先构造通过 check 的 error 取值，再利用少值 error 的结构恢复 secret。 | [lattice-and-lwe.md](lattice-and-lwe.md) |
| [WMCTF2025-splitmaster-wp](../raw/crypto/WMCTF2025-splitmaster-wp.md) | 交互 oracle 允许自选分段泄露 `a*key mod q` 的不连续 bit，先设计切分消去公共隐藏数，再用 BKZ 恢复 key。 | [lattice-and-lwe.md](lattice-and-lwe.md) |
| [WMCTF2025-lemonpepper-wp](../raw/crypto/WMCTF2025-lemonpepper-wp.md) | 重根多项式、p-adic roots 和三选一递推状态混合，先用导数/去重根恢复候选，再用递推剪枝同步 state。 | [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md) |
| [ACTF2026-arrange-in-asceding-wp](../raw/crypto/ACTF2026-arrange-in-asceding-wp.md) | CKKS 排名题要在密文 slot 中打包 128x128 两两比较，靠旋转、Chebyshev 近似和 rescale 控制乘法深度。 | [homomorphic-and-exotic-algebra.md](homomorphic-and-exotic-algebra.md) |
| [ACTF2026-inverse-pow-wp](../raw/crypto/ACTF2026-inverse-pow-wp.md) | `2^n` 十进制前缀匹配可转成 `{n log10(2)}` 命中窄区间，先用固定点 BSGS/LLL 找足够小的 `n`。 | [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md) |
| [ACTF2026-ohmycaptcha-wp](../raw/crypto/ACTF2026-ohmycaptcha-wp.md) | 验证码其实是模素数余数约束；用 CRT+LLL 构造 `eval` 表达式，再利用 `e=5` RSA 小明文/Coppersmith/Franklin-Reiter 恢复 AES-CTR 参数。 | [oracles-recurrences-captcha-polyglots.md](oracles-recurrences-captcha-polyglots.md)、[hash-protocol-and-oracle-attacks.md](hash-protocol-and-oracle-attacks.md)、[rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md)、[lattice-and-lwe.md](lattice-and-lwe.md) |
| [ACTF2026-pandora-wp](../raw/crypto/ACTF2026-pandora-wp.md) | 二次域 ideal / binary quadratic form 泄露判别式小数、`A` 高位和 `B`，先恢复判别式与 `A`，再做二元 Coppersmith 小根。 | [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md) |
| [Bugku-alkane-wp](../raw/crypto/Bugku-alkane-wp.md) | 已知明文 XOR 出 128 bit keystream，`schedule` 把 key bit 线性组合到输出 bit，直接建 GF(2) 线性方程并枚举少量自由变量。 | [rc4-lfsr-and-keystream-reuse.md](rc4-lfsr-and-keystream-reuse.md) |
| [Bugku-apbq-rsa-ii-wp](../raw/crypto/Bugku-apbq-rsa-ii-wp.md) | 三条 `a*p+b*q` 小系数 hint 可构成缩放格，LLL 找短关系后对候选分量与 `n` 取 gcd 分解。 | [lattice-and-lwe.md](lattice-and-lwe.md) |
| [Bugku-Fibonacci-RSA-wp](../raw/crypto/Bugku-Fibonacci-RSA-wp.md) | RSA 素数由 `next_prime(S^16)` 生成，`S` 是 Fibonacci 模小 `n` 的长和，先用 Pisano period 快速恢复 `S`。 | [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md)、[rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md) |
| [HGAME2026-babyrsa-wp](../raw/crypto/HGAME2026-babyrsa-wp.md) | `p,q` 公开但模数小于完整 flag，密文只约束 `m mod n`；结合前后缀、长度和字符集用格恢复原明文字节。 | [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md) |
| [HGAME2026-classic-wp](../raw/crypto/HGAME2026-classic-wp.md) | RSA 泄露 `p >> 230`，直接令 `p=p_high+x` 做一元 Coppersmith；解密后还要按明文提示走 Vigenere 二层。 | [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md)、[lattice-and-lwe.md](lattice-and-lwe.md) |
| [HGAME2026-decision-wp](../raw/crypto/HGAME2026-decision-wp.md) | 每个 flag bit 对应一个 chunk：bit=1 是同一 LWE 的 15 样本，bit=0 是随机；先合并两个真 chunk 格恢复共享 `s`，再按误差大小分类。 | [lattice-and-lwe.md](lattice-and-lwe.md) |
| [HGAME2026-eezzdlp-wp](../raw/crypto/HGAME2026-eezzdlp-wp.md) | 矩阵 DLP 在 `n=p^2` 且 `det(a)=1` 时不能走行列式；改用特征值上的 p-adic log 和 Pohlig-Hellman 补低位。 | [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md) |
| [HGAME2026-ezcurve-wp](../raw/crypto/HGAME2026-ezcurve-wp.md) | ECC 横坐标 oracle 每次减去 163-bit 噪声；成对查询 `t/-t` 消去部分曲线关系，再把小误差放进 LLL。 | [ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md)、[lattice-and-lwe.md](lattice-and-lwe.md) |
| [HGAME2026-ezdlp-wp](../raw/crypto/HGAME2026-ezdlp-wp.md) | 模合数矩阵 DLP 可先取行列式，把 `a^k=b` 降成 `det(a)^k=det(b)`，再分模数做 Pohlig-Hellman 和 CRT。 | [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md) |
| [HGAME2026-ezrsa-wp](../raw/crypto/HGAME2026-ezrsa-wp.md) | 指数 bit-flip 加密 oracle 可恢复 `n` 和 50-bit `e`，拿到 flag 密文后再用解密 oracle 做低字节推进。 | [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md) |
| [HGAME2026-flux-wp](../raw/crypto/HGAME2026-flux-wp.md) | 4 个连续状态满足有限域二次递推，先用 Groebner 消元恢复递推参数；自定义 `shash` 的 key 再用 bit-vector SMT 反求。 | [oracles-recurrences-captcha-polyglots.md](oracles-recurrences-captcha-polyglots.md)、[number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md) |
| [LilacCTF2026-bootstrapping-wp](../raw/crypto/LilacCTF2026-bootstrapping-wp.md) | 同态、配对或特殊代数结构暴露运算关系，先确认可组合操作和明文空间。 | [homomorphic-and-exotic-algebra.md](homomorphic-and-exotic-algebra.md) |
| [LilacCTF2026-myblock-wp](../raw/crypto/LilacCTF2026-myblock-wp.md) | 8 轮 16-bit Feistel 的轮函数是 `GF(2^16)` 上低次多项式；用明密文样本建正向/反向基矩阵，枚举子密钥看秩下降。 | [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md) |
| [LilacCTF2026-myrsa-wp](../raw/crypto/LilacCTF2026-myrsa-wp.md) | 三素数 RSA 的 `p,q` 由同一 `pp` 二次多项式生成，`4p=t^2+3` 暴露 CM 曲线结构；用平方根 oracle 点构造 ECM 型 gcd 分解。 | [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md)、[number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md) |
| [LilacCTF2026-nestdlp-wp](../raw/crypto/LilacCTF2026-nestdlp-wp.md) | 二元多项式商环 DLP 可转成乘法矩阵行列式上的 p-adic DLP；恢复指数后还要用 padding 汉明重量约束解明文。 | [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md)、[oracles-recurrences-captcha-polyglots.md](oracles-recurrences-captcha-polyglots.md) |
| [LilacCTF2026-noisy-forest-wp](../raw/crypto/LilacCTF2026-noisy-forest-wp.md) | Python MT bitstream 只以“中文字符是否 +9997”泄露；先用中文文本冗余恢复前缀，再从明密文差异克隆 MT19937 状态。 | [mt-lcg-and-seed-recovery.md](mt-lcg-and-seed-recovery.md) |
| [NCTF2026-encryption-wp](../raw/crypto/NCTF2026-encryption-wp.md) | pwn 后 dump `libcipher.so`，发现魔改 AES 的 S-box 被线性化；把整块加密建成 GF(2) 仿射变换矩阵再求逆。 | [block-mode-misuse-family.md](block-mode-misuse-family.md)、[number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md) |
| [NCTF2026-ez-rsa-wp](../raw/crypto/NCTF2026-ez-rsa-wp.md) | hint 同时约束 `p/q` 高低位；用双端 DFS 维护乘积上下界和低位乘积模约束，恢复因子后常规解密。 | [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md) |
| [NCTF2026-hard-rsa-wp](../raw/crypto/NCTF2026-hard-rsa-wp.md) | RSA-like 体制使用 `phi_6(N)=(p^6-1)(q^6-1)`；先连分数找 generalized Wiener 候选，再由 `p_hat` 小误差转 Coppersmith/近邻分解。 | [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md)、[lattice-and-lwe.md](lattice-and-lwe.md) |
| [NCTF2026-rng-game-wp](../raw/crypto/NCTF2026-rng-game-wp.md) | 服务给出 Python `random` 大整数 seed，目标是构造另一 seed；按 32-bit 分块利用 CPython seed 扩展阶段碰撞。 | [mt-lcg-and-seed-recovery.md](mt-lcg-and-seed-recovery.md) |
| [NCTF2026-yqs-wp](../raw/crypto/NCTF2026-yqs-wp.md) | 先用 LWE primal attack 恢复参与坐标 XOR 的私钥，再构造无效曲线小阶点，多轮 CRT 合并 master secret。 | [lattice-and-lwe.md](lattice-and-lwe.md)、[ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md) |
| [RCTF2025-f-l-and-ag-plusplus-wp](../raw/crypto/RCTF2025-f-l-and-ag-plusplus-wp.md) | 高次数拼接关系需要快速 resultant、CRT basis 和 Lagrange interpolation；恢复比例后再 rational reconstruction 拼回 flag。 | [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md)、[lattice-and-lwe.md](lattice-and-lwe.md) |
| [RCTF2025-repairing-wp](../raw/crypto/RCTF2025-repairing-wp.md) | Pairing/ElGamal 风格密文可重随机化：同时改 `C1,C2,C3` 保持 shared key 不变，服务端未去重时可解出原 flag key。 | [ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md)、[homomorphic-and-exotic-algebra.md](homomorphic-and-exotic-algebra.md) |
| [RCTF2025-suanhash-wp](../raw/crypto/RCTF2025-suanhash-wp.md) | Sponge-like hash 输出泄露 rate 和 capacity 派生差分；查询两条近似消息恢复内部差分，再追加互补块制造碰撞。 | [hash-protocol-and-oracle-attacks.md](hash-protocol-and-oracle-attacks.md) |
| [RCTF2025-suanp01y-wp](../raw/crypto/RCTF2025-suanp01y-wp.md) | 稀疏多项式满足 `hint = h1 * h0^{-1}` 于 `GF(2)[X]/(X^r-1)`；做 rational reconstruction 恢复分母，再枚举循环位移解 AES-CTR。 | [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md) |
| [RCTF2025-superguess-plusplus-wp](../raw/crypto/RCTF2025-superguess-plusplus-wp.md) | HNP 参数极紧：约 93 个样本、每个只泄露 nonce MSB 2 bit；需改 Fourier/lattice-sieving HNP solver 并调整样本/线程/校验。 | [lattice-and-lwe.md](lattice-and-lwe.md) |
| [RCTF2025-yet-another-mt-game-wp](../raw/crypto/RCTF2025-yet-another-mt-game-wp.md) | Sage `random_matrix(Zmod(mod))` 小模数路径泄露 GMP MT 31-bit 输出；用 GF(2) 线性系统恢复状态，再反推 GMP seed。 | [mt-lcg-and-seed-recovery.md](mt-lcg-and-seed-recovery.md) |
| [RCTF2025-yet-another-shuffled-mt-game-wp](../raw/crypto/RCTF2025-yet-another-shuffled-mt-game-wp.md) | GMP MT 输出被 Python `random.shuffle` 置乱；先从少量输出恢复 128-bit Python seed 逆置乱，再复用 GMP MT 状态恢复。 | [mt-lcg-and-seed-recovery.md](mt-lcg-and-seed-recovery.md) |
| [SU_AESWP](../raw/crypto/SU_AESWP.md) | AES 服务允许分别更新 seed/key，导致当前 S-box 与旧 round keys 不同步；先把 S-box 压成常值恢复 `K10`，再用 probe 值域指纹恢复置换。 | [block-mode-misuse-family.md](block-mode-misuse-family.md) |
| [SU_IsogenyWP](../raw/crypto/SU_IsogenyWP.md) | CSIDH 群作用与 2-isogeny 交换，三条 2-isogenous 共享曲线高位泄露可建 CI-HNP 小根方程，用 Automated Coppersmith 恢复参数。 | [ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md)、[lattice-and-lwe.md](lattice-and-lwe.md) |
| [SU_LatticeWP](../raw/crypto/SU_LatticeWP.md) | 24 阶 Fibonacci LFSR 只泄露状态高 40 bit，且模数未知但靠近 2 的幂；用高位格找 annihilating polynomials、resultant gcd 恢复模数，再 BKZ 低位状态。 | [lattice-and-lwe.md](lattice-and-lwe.md)、[rc4-lfsr-and-keystream-reuse.md](rc4-lfsr-and-keystream-reuse.md) |
| [SU_PrngWP](../raw/crypto/SU_PrngWP.md) | 256-bit LCG 输出被“高低半 XOR + 按高位 ror”非线性包装；先恢复 rotation sequence/低半候选，再用 LCG 关系约束原 seed。 | [mt-lcg-and-seed-recovery.md](mt-lcg-and-seed-recovery.md) |
| [SU_RestaurantWP](../raw/crypto/SU_RestaurantWP.md) | Tropical semiring 验证只检查最终矩阵等式和 rank/range；可构造目标矩阵 `T`，再分别构造 `A/B/P/R/S` 压住未知 `fork` 项。 | [homomorphic-and-exotic-algebra.md](homomorphic-and-exotic-algebra.md) |
| [SU_RSAWP](../raw/crypto/SU_RSAWP.md) | 小 `d` 与 `p+q` 高位泄露同时存在；把 `ed=k(N-S-x+1)+1` 写成二元小根，修正 lattice shift 后用 Resultant 提低位 `x`。 | [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md)、[lattice-and-lwe.md](lattice-and-lwe.md) |
| [VNCTF2026-ezov-wp](../raw/crypto/VNCTF2026-ezov-wp.md) | UOV/OV 公钥是多组二次型矩阵，中心 oil-oil 块为 0；先恢复等价 vinegar/oil 子空间，再对 `admin` 固定 vinegar 求 oil 伪造签名。 | [homomorphic-and-exotic-algebra.md](homomorphic-and-exotic-algebra.md) |
| [VNCTF2026-hd-is-what-wp](../raw/crypto/VNCTF2026-hd-is-what-wp.md) | SIDH/SIKE 公钥向量先被公开 seed 的 LCG 矩阵线性混淆；恢复标准公钥后再用 Castryck-Decru attack 求共享 j。 | [ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md)、[mt-lcg-and-seed-recovery.md](mt-lcg-and-seed-recovery.md) |
| [VNCTF2026-math-rsa-wp](../raw/crypto/VNCTF2026-math-rsa-wp.md) | 泄露式可因式分解为 `(x+1)^2(y-1)^2=4k`，其中 `x=phi-1`、`y=t+1`；枚举 16-bit `t/2` 即可恢复 `phi`。 | [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md)、[number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md) |
| [VNCTF2026-numberguesser-wp](../raw/crypto/VNCTF2026-numberguesser-wp.md) | 只有 10 次 hint 查询，但 Python `random.seed(os.urandom(8))` 可逆；选相隔 227 的输出 untemper/twist 反推 64-bit seed。 | [mt-lcg-and-seed-recovery.md](mt-lcg-and-seed-recovery.md) |
| [VNCTF2026-schnorr-wp](../raw/crypto/VNCTF2026-schnorr-wp.md) | Schnorr 服务固定 seed 导致首轮承诺 `B` 跨连接重复；对同一 `B` 给两个 challenge，用 special soundness 相减提 witness。 | [ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md)、[mt-lcg-and-seed-recovery.md](mt-lcg-and-seed-recovery.md) |
| [D3CTF2019-babyecc-wp](../raw/crypto/D3CTF2019-babyecc-wp.md) | 曲线参数和 ECC 群运算是核心，先确认曲线阶、基点和标量关系。 | [ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md) |
| [D3CTF2019-bivariate-wp](../raw/crypto/D3CTF2019-bivariate-wp.md) | RSA 素数中间位泄露可写成二元小根，先估计未知高低位界并构造 Coppersmith 格。 | [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md) |
| [D3CTF2019-common-wp](../raw/crypto/D3CTF2019-common-wp.md) | 共模 RSA 小私钥指数变体，先把 Wiener/Guo 方程转成低维格再恢复 phi。 | [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md) |
| [D3CTF2019-noise-wp](../raw/crypto/D3CTF2019-noise-wp.md) | 带噪声取模 oracle 需要维护候选区间，构造固定商查询把随机噪声转成强收缩。 | [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md) |
| [D3CTF2019-sign2win-wp](../raw/crypto/D3CTF2019-sign2win-wp.md) | DSA/ECDSA 签名或 nonce 关系异常，先恢复签名方程中的随机数/私钥约束。 | [ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md) |
| [D3CTF2021-alice-want-flag-wp](../raw/crypto/D3CTF2021-alice-want-flag-wp.md) | ElGamal 乘法同态和长度 oracle 可逐位泄露密码，再结合短 key meet-in-the-middle。 | [homomorphic-and-exotic-algebra.md](homomorphic-and-exotic-algebra.md) |
| [D3CTF2021-baby-lattice-simple-group-wp](../raw/crypto/D3CTF2021-baby-lattice-simple-group-wp.md) | 小矩阵/RSA 模数上的线性关系可转成低维格，先恢复小等价表示或秘密比值。 | [lattice-and-lwe.md](lattice-and-lwe.md) |
| [D3CTF2021-easy-curve-wp](../raw/crypto/D3CTF2021-easy-curve-wp.md) | 曲线/DLP 参数是主约束，先确认群阶、基点、映射和可降维关系。 | [ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md) |
| [D3CTF2022-d3bug-wp](../raw/crypto/D3CTF2022-d3bug-wp.md) | LFSR 输出直接泄露移出位，剩余低位可转成 GF(2) 线性方程或 SAT/Z3。 | [mt-lcg-and-seed-recovery.md](mt-lcg-and-seed-recovery.md) |
| [D3CTF2022-d3factor-wp](../raw/crypto/D3CTF2022-d3factor-wp.md) | Prime Power RSA 的指数差满足小根方程，先在 p 的高次幂模意义下构造 Coppersmith。 | [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md) |
| [D3CTF2022-d3qcg-wp](../raw/crypto/D3CTF2022-d3qcg-wp.md) | 二次同余生成器只泄露高位，先把未知低位建成二元小根再恢复 PRNG 状态。 | [mt-lcg-and-seed-recovery.md](mt-lcg-and-seed-recovery.md) |
| [D3CTF2022-d3share-wp](../raw/crypto/D3CTF2022-d3share-wp.md) | 扰动多项式密钥共享可从多节点私钥插值和格约束中恢复等效主多项式。 | [exotic-secret-sharing-rabin-and-polynomials.md](exotic-secret-sharing-rabin-and-polynomials.md) |
| [D3CTF2022-equivalent-wp](../raw/crypto/D3CTF2022-equivalent-wp.md) | Knapsack/子集和公钥可构造等效私钥，先取正交格再找满足奇偶和大小条件的短向量。 | [lattice-and-lwe.md](lattice-and-lwe.md) |
| [D3CTF2022-leak-dsa-wp](../raw/crypto/D3CTF2022-leak-dsa-wp.md) | DSA/ECDSA nonce 或签名泄露进入格恢复，先写出每个签名的线性近似关系。 | [ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md) |
| [D3CTF2023-d3bdd-wp](../raw/crypto/D3CTF2023-d3bdd-wp.md) | LWE/RLWE dual attack 和理想格短向量是主线，先检查 PRNG 与模多项式是否破坏格结构。 | [lattice-and-lwe.md](lattice-and-lwe.md) |
| [D3CTF2023-d3noisy-wp](../raw/crypto/D3CTF2023-d3noisy-wp.md) | 打乱余数导致 noisy CRT，先转成子集和/背包匹配，再用格规约恢复大整数。 | [lattice-and-lwe.md](lattice-and-lwe.md) |
| [D3CTF2023-d3pack-wp](../raw/crypto/D3CTF2023-d3pack-wp.md) | Affine hidden subset sum 需要 completion/orthogonal lattice，先找 0/1 或 -1/0/1 短向量。 | [lattice-and-lwe.md](lattice-and-lwe.md) |
| [D3CTF2023-d3sys-wp](../raw/crypto/D3CTF2023-d3sys-wp.md) | CTR-SM4 token 可篡改后进入 CRT-RSA partial key exposure，重点估泄露位和 Coppersmith bound。 | [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md) |
| [D3CTF2025-d3fnv-wp](../raw/crypto/D3CTF2025-d3fnv-wp.md) | FNV 哈希可视为未知 key 上的小系数多项式求值，先用格恢复 key 幂向量信息。 | [hash-protocol-and-oracle-attacks.md](hash-protocol-and-oracle-attacks.md) |
| [D3CTF2025-d3guess-wp](../raw/crypto/D3CTF2025-d3guess-wp.md) | 带噪声猜数反馈要用概率二分收集输出，再恢复 MT19937 状态预测后续随机数。 | [mt-lcg-and-seed-recovery.md](mt-lcg-and-seed-recovery.md) |
| [D3CTF2025-d3sys-p2-2-0-wp](../raw/crypto/D3CTF2025-d3sys-p2-2-0-wp.md) | RSA/CRT 因子有已知线性近似且修正项很小，优先建 partial factoring 小根模型。 | [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md) |

## 常见陷阱

- 把 WP 当完整题解目录使用，没有提取可迁移的触发信号。
- 只按题名命中下一跳，忽略源码、参数规模、可控输入和错误反馈。
- 已有技巧页能覆盖时仍新建单题页面，导致 wiki 退化成 writeup 列表。

## 关联技巧

- [block-mode-misuse-family.md](block-mode-misuse-family.md)
- [ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md)
- [exotic-secret-sharing-rabin-and-polynomials.md](exotic-secret-sharing-rabin-and-polynomials.md)
- [hash-protocol-and-oracle-attacks.md](hash-protocol-and-oracle-attacks.md)
- [homomorphic-and-exotic-algebra.md](homomorphic-and-exotic-algebra.md)
- [lattice-and-lwe.md](lattice-and-lwe.md)
- [mt-lcg-and-seed-recovery.md](mt-lcg-and-seed-recovery.md)
- [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md)
- [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md)
- [zkp-secret-sharing-and-proof-systems.md](zkp-secret-sharing-and-proof-systems.md)

## 原始资料

本页原始资料以“WP 案例沉淀”表中的 raw WP 链接为准，共 69 篇。
