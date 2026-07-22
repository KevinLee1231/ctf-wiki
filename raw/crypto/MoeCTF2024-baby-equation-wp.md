# baby_equation

## 题目简述

flag 被等分成两半并转为整数 $a,b$，题目只给出一个大整数 $k$ 和一条看似复杂的四次方程。关键是先把方程化为完全平方，再把 $2\sqrt{k}$ 的素因子分配给 $a+1$ 与 $b-1$。

## 解题过程

源码给出的关系是：

$$
(a^2+1)(b^2+1)-2(a-b)(ab-1)=4(k+ab).
$$

把右侧的 $4ab$ 移到左侧并因式分解：

$$
(a^2+1)(b^2+1)-2(a-b)(ab-1)-4ab
=(a+1)^2(b-1)^2.
$$

因此：

$$
(a+1)^2(b-1)^2=4k,
$$

而 $a,b$ 均为正整数，所以

$$
(a+1)(b-1)=2\sqrt{k}.
$$

实际 $k$ 是完全平方数。对 $2\sqrt{k}$ 分解后得到 18 个含重数的素因子，枚举其子集作为 $a+1$，余下部分自然是 $b-1$：

```python
from math import isqrt, prod
from Crypto.Util.number import long_to_bytes

k = 0x2227e398fc6ffcf5159863a345df85ba50d6845f8c06747769fee78f598e7cb1bcf875fb9e5a69ddd39da950f21cb49581c3487c29b7c61da0f584c32ea21ce1edda7f09a6e4c3ae3b4c8c12002bb2dfd0951037d3773a216e209900e51c7d78a0066aa9a387b068acbd4fb3168e915f306ba40

root = isqrt(k)
assert root * root == k
target = 2 * root

factors = [
    2, 2, 2, 2, 3, 3, 31, 61, 223, 4013, 281317, 4151351,
    26798471753993, 339386329, 25866088332911027256931479223,
    370523737, 5404604441993,
    64889106213996537255229963986303510188999911,
]
assert prod(factors) == target

for mask in range(1 << len(factors)):
    a_plus_one = prod(
        factor for index, factor in enumerate(factors)
        if (mask >> index) & 1
    )
    b_minus_one = target // a_plus_one
    candidate = (
        long_to_bytes(a_plus_one - 1)
        + long_to_bytes(b_minus_one + 1)
    )
    if candidate.startswith(b"moectf{") and candidate.endswith(b"}"):
        print(candidate)
        break
```

输出：

```text
moectf{7he_Fund4m3nt4l_th30r3m_0f_4rithm3tic_i5_p0w4rful!}
```

## 方法总结

这题先用代数恒等式把双变量方程降为因子分配问题，再利用 flag 格式筛选正确拆分。原稿展示的因子列表与实际 $k$ 不一致，循环也漏用了因子；这里已按源码重新分解，并验证因子乘积恰为 $2\sqrt{k}$。
