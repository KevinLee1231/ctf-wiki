# One more bit

## 题目简述

题目生成约 1024 位的 RSA 模数，却把私钥指数 $d$ 固定为恰好 258 位。经典 Wiener 攻击通常要求

$$
d<\frac{1}{3}N^{1/4},
$$

对 1024 位模数而言大致落在 256 位附近，因此本题故意比直接攻击的范围多出少量比特。需要在连分数收敛分母的基础上再搜索小系数组合，恢复 $d$ 后解密密文。

## 解题过程

RSA 满足

$$
ed-k\varphi(N)=1,
$$

所以当 $d$ 足够小时，$k/d$ 会非常接近 $e/N$。Wiener 攻击枚举 $e/N$ 的连分数收敛分数，并把其分母作为 $d$ 的候选。[Boneh 的 RSA 攻击综述](https://www.ams.org/notices/199902/boneh.pdf)给出了该攻击的条件与推导：在素数规模接近且 $d<N^{1/4}/3$ 时，可以从公开的 $(N,e)$ 高效恢复私钥指数。

这里的 $d$ 有 258 位，已经越过经典界限，但只多出很小范围。[关于 Wiener 攻击变体的论文](https://arxiv.org/abs/0811.0063)总结了 Verheul–van Tilborg 扩展：真实私钥可以在相邻收敛分母 $q_{i-1},q_i$ 的小系数线性组合

$$
d=rq_i+sq_{i-1}
$$

附近搜索。本实例中枚举 $0\le r,s<20$ 即可覆盖正确候选。

源码还提供了一个比浮点幂估计更可靠的筛选条件：`d.bit_length() == 258`，即

$$
2^{257}\le d<2^{258}.
$$

用这个精确整数区间过滤并去重候选，再以公开指数加密测试消息 `3`，检查候选解密后是否仍为 `3`。命中后用 $d$ 解密题目密文。

另一个容易遗漏的细节是源码连续调用了两次 `pad(..., 16)`：

```python
message = pad(flag, 16)
msg = pad(message, 16)
```

因此解密后必须执行两次 `unpad`。原始解法只去掉外层填充，输出末尾残留的 `\x02\x02` 正是内层填充。

```python
from Crypto.Util.number import long_to_bytes
from Crypto.Util.Padding import unpad
from sage.all import Integer, continued_fraction

n = Integer(
    134133840507194879124722303971806829214527933948661780641814514330769296658351734941972795427559665538634298343171712895678689928571804399278111582425131730887340959438180029645070353394212857682708370490223871309129948337487286534021548834043845658248447393803949524601871557448883163646364233913283438778267
)
e = Integer(
    83710839781828547042000099822479827455150839630087752081720660846682103437904198705287610613170124755238284685618099812447852915349294538670732128599161636818193216409714024856708796982283165572768164303554014943361769803463110874733906162673305654979036416246224609509772196787570627778347908006266889151871
)
ciphertext = Integer(
    73228838248853753695300650089851103866994923279710500065528688046732360241259421633583786512765328703209553157156700672911490451923782130514110796280837233714066799071157393374064802513078944766577262159955593050786044845920732282816349811296561340376541162788570190578690333343882441362690328344037119622750
)

d_min = Integer(1) << 257
d_max = Integer(1) << 258
candidates = set()
previous = Integer(1)

for convergent in continued_fraction(e / n).convergents():
    current = convergent.denominator()
    for r in range(20):
        for s in range(20):
            candidate = r * current + s * previous
            if d_min <= candidate < d_max:
                candidates.add(Integer(candidate))
    previous = current
    if previous >= d_max:
        break

probe = pow(3, e, n)
for d in sorted(candidates):
    if pow(probe, d, n) != 3:
        continue
    padded_twice = long_to_bytes(pow(ciphertext, d, n))
    try:
        plain = unpad(unpad(padded_twice, 16), 16)
    except ValueError:
        continue
    if plain.startswith(b"moectf{"):
        print(d)
        print(plain.decode())
        break
```

最终得到：

```text
moectf{Ju5t_0n3_st3p_m0r3_th4n_wi3n3r_4ttack!}
```

## 方法总结

本题考查的是“略高于 Wiener 界”的小私钥 RSA。先利用连分数给出的高质量逼近，再枚举相邻收敛分母的小系数组合；结合源码明确给出的 258 位范围，可以避免不可靠的浮点边界计算并显著减少候选。恢复 $d$ 之后仍要回到加密源码核对消息预处理，本题的两层 PKCS#7 填充必须对应两次去填充。
