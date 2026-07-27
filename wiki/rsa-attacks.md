---
type: family
tags: [crypto, family, rsa, textbook-rsa]
skills: [ctf-crypto]
raw:
  - ../raw/crypto/rsa-attacks.md
updated: 2026-07-27
---

# RSA Attacks

## 作用边界

本页是 RSA 基础路线 family，用于在看到 `n/e/c`、多组密文、多个模数、弱私钥指数、低指数、可查询 oracle 或可疑 prime 结构时，先判断该走哪条最短路线。它不再作为单一 technique，因为 raw 覆盖的是一组 RSA 攻击入口。

更长尾的素数结构、签名同态、fault、ROCA、CRT 泄露和复杂 oracle 转到 [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md)。

## 识别信号

- 题目给出 `n/e/c`、PEM、公钥集合、多组密文、相同明文提示、低指数、可疑小 `d` 或可查询解密/验签接口。
- 模数之间可能共享因子，素数可能太近、太小、多素数或能被 FactorDB/批量 gcd 直接排查。
- 明文结构有已知前缀、短消息、无 padding、相同消息多模数或同模多指数。
- 响应差异看起来像 padding/timing/oracle 时，应先确认是否已超出基础页，转 specialized。

## 最小证据

- 列出所有公开参数：`n`、`e`、`c`、模数数量、密文数量、是否同明文、是否同模数。
- 先完成低成本检查：FactorDB/小因子、pairwise gcd、近似平方根、`m^e < n`、CRT/Hastad 条件和 Wiener 粗界。
- 对低指数/广播攻击，要证明合并后的明文界满足整数开根。
- 对 oracle 类 RSA，要保存可重放查询和响应分类，再转 specialized 页面继续建模。

## 首轮路由

| 信号 | 最小证据 | 下一跳 |
|---|---|---|
| `m^e < n`、`e=3/5/7`、无 padding | 直接整数开 e 次根能复算密文 | 本页 raw；必要时用 [crypto-tooling.md](crypto-tooling.md) |
| 同一 `n` 下多指数加密同一明文 | `gcd(e1,e2)=1`，有同模密文 | 本页 raw |
| 多个互素模数、同一明文、同一小 `e` | 密文数不少于 `e`，可 CRT 合并 | 本页 raw；线性 padding 转 [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md) |
| 小私钥指数 | `d < n^0.25` 或参数接近 Wiener/continued fraction 条件 | [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md) |
| 连续素数、近似素数、多素数 | `p≈q`、prime count 多、FactorDB/小因子有迹象 | 本页 raw；结构化 prime 转 specialized |
| OAEP/PKCS#1 padding oracle 或 timing oracle | 查询响应能区分区间、padding 或 timing | [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md) |
| 题目变成多项式、GF(2)[x]、仿射模数 | RSA 只是外壳，核心是代数结构 | [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md) |

## Technique 下一跳

| 首轮判断 | 具体 technique |
|---|---|
| 小公开指数、同模多指数、多模同消息或短相关消息 | [rsa-low-exponent-and-related-message-attacks.md](rsa-low-exponent-and-related-message-attacks.md) |
| 共享/相近/结构化素因子，或泄露 `d/dp/dq/phi` 的全部或部分位 | [rsa-factor-relation-and-partial-key-recovery.md](rsa-factor-relation-and-partial-key-recovery.md) |
| 解密响应泄露 padding、区间、奇偶性或 timing | [rsa-padding-and-interval-oracle-attacks.md](rsa-padding-and-interval-oracle-attacks.md) |

## 合并与拆分结论

- 保留为 family：它承接 RSA 基础攻击的首轮判断。
- 不与 `rsa-specialized-structures-and-oracles.md` 合并：基础路线强调快速排查，specialized 页强调复杂结构、oracle 和长尾案例。
- 已将低指数/相关消息、因子关系/部分密钥和 padding/区间 oracle 拆为 technique；本页只保留参数级首轮分流。

## 常见误判

- 只看到 `e=65537` 就放弃 RSA；仍需检查 shared prime、prime structure、CRT 泄露和 oracle。
- 低指数攻击未先估 `m^e < n` 或 CRT 合并后的界。
- Manger/Bleichenbacher 类 oracle 没量化响应差异，导致查询脚本不可复现。
- 先跑重型 Sage/LLL，而没有做 FactorDB、gcd、近似平方根和参数规模首检。

## 关联页面

- [crypto-parameter-triage-family.md](crypto-parameter-triage-family.md)
- [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md)
- [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md)
- [lattice-and-lwe.md](lattice-and-lwe.md)
- [hash-protocol-and-oracle-attacks.md](hash-protocol-and-oracle-attacks.md)
- [crypto-tooling.md](crypto-tooling.md)

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [0xGame2022-week2-re2-wp](../raw/reverse/0xGame2022-week2-re2-wp.md) | Go `math/big` 调用链是 RSA 校验；重点是提取 `n/c/e` 并注意 `SetString` 的 base 参数，密文是八进制。 |
| [0xGame2020-week4-littleTrick-wp](../raw/crypto/0xGame2020-week4-littleTrick-wp.md) | 漏洞由“可控后缀替换 + 确定性裸 RSA”共同造成。调整 mask 长度即可让响应中只剩一个未知字节，再进行本地加密比对。实现时必须审计 `long_to_bytes` 的零值行为：该接口无法暴露最后一个原始字节，本题只能依据已知 flag 格式补齐结尾。 |
| [0xGame2021-week3-Fermat-with-Binomial-wp](../raw/crypto/0xGame2021-week3-Fermat-with-Binomial-wp.md) | 含 $p,q$ 的 RSA hint 应分别在模 $p$、模 $q$ 下化简。二项式展开可消去含某个素因子的项，Fermat 同余可降低指数；构造出“模某素因子为零”的表达式后，用 `gcd` 提取该因子。 |
| [0xGame2021-week3-Wilson-wp](../raw/crypto/0xGame2021-week3-Wilson-wp.md) | 相邻素数构成的 RSA 模数适合 Fermat 分解。面对无法直接计算的超大阶乘，应寻找 Wilson 定理等模意义恒等式；本题只需计算很短的区间乘积 $p\cdots(q-1)$，即可消去 $(p-1)!$。 |
| [0xGame2022-week4-我也不知道取啥名捏-wp](../raw/crypto/0xGame2022-week4-我也不知道取啥名捏-wp.md) | RSA 要求两个素数彼此独立。这里虽然 $p$ 本身是强素数，但由其按位取反派生 $q$，等于泄露了两者的近似和，使分解退化为求二次方程近似根并搜索很小的低位误差。检查自定义密钥生成器时，应优先寻找因子之间的和、差、异或或共享位关系。 |
| [0xGame2023-week4-Drange-Leak-wp](../raw/crypto/0xGame2023-week4-Drange-Leak-wp.md) | 泄露量 $M$ 的价值在于模 $eM$ 后可以消去未知的 $d_1$，只留下三个有界未知量 $d_0$、$k$ 和 $p+q-1$。多元 Coppersmith/LLL 负责恢复小根；拿到 $p+q$ 后应立即用二次方程验证并分解 $n$。参数 `bounds` 是格缩放的一部分，必须区分严格数学上界与为了规约效果采用的经验值。 |
| [0xGame2025-week2-PolyRSA-wp](../raw/crypto/0xGame2025-week2-PolyRSA-wp.md) | 把 RSA 的“指数求逆”推广到有限商环，并选择单位群指数的可用公倍数；消息被编码为多项式、运算发生在 $\mathbb Z_n[x]/(g(x))$、且 $p,q$ 已知。 |
| [0xGame2025-week3-New-Python-wp](../raw/crypto/0xGame2025-week3-New-Python-wp.md) | 本题把 RSA CRT 参数泄露、HTTP 信息搜集和 UUIDv8 字段布局串联起来。看到 $d_p$ 时应立即利用 $ed_p-1$ 是 $p-1$ 倍数这一关系分解 $n$；得到明文后，还要检查响应头和隐藏路由补齐其余参数。生成 UUID 时需注意字段会截取低位，并区分“Python 3.14”与不存在的“Python 14”。 |
| [ACTF2022-rsa-leak-wp](../raw/crypto/ACTF2022-rsa-leak-wp.md) | 本题的泄露不是直接给出 $p$ 或 $q$ 的比特，而是把两个小偏移的模幂相加。中途相遇先把 $2^{48}$ 的联合搜索降为约 $2^{24}$；随后利用 $n$ 紧邻完全四次幂得到 $ab$，再把剩余关系整理成关于 $a^4$ 的二次方程，精确恢复两个素因子。 |
| [D3CTF2024-myRSA-wp](../raw/crypto/D3CTF2024-myRSA-wp.md) | 由 LFSR 周期确定两个 RSA 素数在不同偏移处共享长 bit 段，再使用 GIFP 的多元 Coppersmith/LLL 攻击分解模数；随机源输出长度超过周期、两个模数的素数片段出现错位重叠时，不能只套用“同位置共享高低位”的 IFP，应建立带偏移变量的广义模型。 |
| [MoeCTF2021-ezrsa-wp](../raw/crypto/MoeCTF2021-ezrsa-wp.md) | 单调多项式逆像、由 $ed\equiv1\pmod{\varphi(n)}$ 构造 GCD 因子，以及通过降低 `gift` 的模数保留一阶二项式项；RSA 辅助量同时出现 $(p-s)d$、多素数模数和 $(qr+1)^q$ 时，应分别检查指数同余与模 $N^2$ 的二项式展开。 |
| [MoeCTF2023-factorize-me-wp](../raw/crypto/MoeCTF2023-factorize-me-wp.md) | 公开 $\varphi(N)$ 不只意味着可以计算 RSA 私钥，它还给出了模 $N$ 乘法群的阶，可通过寻找非平凡平方根直接分解多素数模数。本题第二层的关键细节是 `nextprime` 映射和有放回抽样：先分解公开的大模数，再映射、试除，并正确处理重复素因子。 |
| [MoeCTF2024-大白兔-wp](../raw/crypto/MoeCTF2024-大白兔-wp.md) | RSA 中出现 $ap+bq$ 的幂泄漏时，应先利用模 $pq$ 后混合项消失的性质。不同指数可通过交叉乘幂齐次化，再用系数组合消去 $p^E$ 或 $q^E$；最终 GCD 把“模意义下含某个素因子”的信息转成实际因子。 |
| [MoeCTF2024-babe-Lifting-wp](../raw/crypto/MoeCTF2024-babe-Lifting-wp.md) | 低位泄漏先通过 $ed-1=k\varphi(n)$ 转成素因子低位，再把未知高位视为小根。Coppersmith 的作用和参数界已在正文中给出，原稿引用的通用个人博客不再是复现所必需，因此删除该外链。 |
| [MoeCTF2024-One-more-bit-wp](../raw/crypto/MoeCTF2024-One-more-bit-wp.md) | 本题考查的是“略高于 Wiener 界”的小私钥 RSA。先利用连分数给出的高质量逼近，再枚举相邻收敛分母的小系数组合；结合源码明确给出的 258 位范围，可以避免不可靠的浮点边界计算并显著减少候选。恢复 $d$ 之后仍要回到加密源码核对消息预处理，本题的两层 PKCS#7 填充必须对应两次去填充。 |
| [MoeCTF2024-rsa-revenge-wp](../raw/crypto/MoeCTF2024-rsa-revenge-wp.md) | 结构化素数会破坏 RSA 对“难以从 $n$ 恢复 $p,q$”的依赖。本题无需通用整数分解：位反转关系把每层搜索限制为四个分支，而低位乘法同余会立即淘汰大多数候选；恢复到中点时两数已完全确定。 |
| [SekaiCTF2026-apbq-rsa-iv-wp](../raw/crypto/SekaiCTF2026-apbq-rsa-iv-wp.md) | 线性组合中同时出现两个大素数并不意味着泄露安全；真正决定难度的是系数界。小系数使多个泄露式之间产生可由格约简捕获的短关系，而恒等式；$h^2-4abN=(ap-bq)^2$ 又把短关系转化为可验证的完全平方条件。 |
| [UMDCTF2018-broken-rsa-wp](../raw/crypto/UMDCTF2018-broken-rsa-wp.md) | RSA 题不能只看文件格式；必须核对 $n$ 是否为合数、$e$ 是否合理，以及 $\gcd(e,\varphi(n))$ 是否为 $1$。本题的决定性错误是把素数直接当成 RSA 模数，使问题退化为有限域开方。 |
| [UMDCTF2020-relatively-rough-rsa-wp](../raw/crypto/UMDCTF2020-relatively-rough-rsa-wp.md) | 本题有两个独立弱点：近素数使 $n$ 可被 Fermat 快速分解，偶数指数又把末步变成 Rabin 平方根问题。不能把 $e$ 当作普通 RSA 指数强行求逆；正确做法是先逆掉 $e/2$，再枚举平方根。 |
| [UMDCTF2023-eeveelutions-wp](../raw/crypto/UMDCTF2023-eeveelutions-wp.md) | 先用 Coppersmith 短填充攻击恢复两条明文的小差值，再用 Franklin-Reiter 相关消息攻击求出原文；同一 RSA 模数、$e=3$、两条高度相似明文和很短的独立后缀同时出现时，应检查相关消息而非尝试分解 $n$。 |
| [UMDCTF2024-key-recovery-wp](../raw/crypto/UMDCTF2024-key-recovery-wp.md) | 把截断 PEM 转成 RSA 参数的已知位/未知位模型，用 $n=pq$、$ed-1=k\lambda(n)$ 和 CRT 指数关系传播比特，最后以低维 LLL 收尾；私钥文件虽然损坏，但 DER 字段边界仍可定位，且多个 RSA 私钥参数同时部分泄露时，不应只盯着模数分解。 |
| [WMCTF2020-meet-in-july-wp](../raw/crypto/WMCTF2020-meet-in-july-wp.md) | 从程序中提取 HMAC 参数和 Lucas 指数 `e=7`，分解小模数 `N`，用 `lcm(p-1,q-1,p+1,q+1)` 构造 Lucas-RSA 私钥指数反解；校验函数不是普通 `pow(s,e,N)`，而是递推 Lucas 序列 `V_i`，同时模数很小可分解，应联想到 Lucas/RSA 变体。 |
| [WMCTF2020-piece-of-cake-wp](../raw/crypto/WMCTF2020-piece-of-cake-wp.md) | 类 NTRU 关系先给出小私钥候选，经可验证输出筛选后，用多组共享指数 RSA 构造格并 LLL 恢复 $e$；最后由 $ed \equiv 1 \pmod{\varphi(N)}$ 推出 $2^{ed-1} \equiv 1 \pmod p$，通过 GCD 分解模数。 |
| [WMCTF2023-signin-wp](../raw/crypto/WMCTF2023-signin-wp.md) | 先利用异或泄露和乘积低位约束分解 RSA 模数，再把乘法模方程的低位泄露转成 HNP 格问题；若题目泄露 `P ^ (Q >> offset)` 这类交错位关系，可以从低位逐步扩展候选；若泄露多组 `a_i*x mod p` 的低位，可考虑 LLL。 |

## 原始资料

- [rsa-attacks.md](../raw/crypto/rsa-attacks.md)
