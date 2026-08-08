# MiniLCTF2023 - not RSA

## 题目简述

题目实现的是基于三次 Pell 方程的 Murru–Saettone 类 RSA 密码系统。模数仍为 $n=pq$，但群阶改为

$$
\Phi=(p^2+p+1)(q^2+q+1),
$$

私钥指数 `d` 只有 276 位，公钥满足 $ed\equiv1\pmod\Phi$。附件还泄露了 `p0 = p + tmp`，其中 `tmp` 为 469 位素数。核心任务是利用小私钥指数和这个近似因子恢复 $p,q,d$，再用题目自定义群运算解密两个消息分量。

## 解题过程

把 $S=p+q$ 代入群阶可得

$$
\Phi=S^2+(n+1)S+n^2-n+1.
$$

`p0` 给出了 $p$ 的近似值，因此可以构造论文改进攻击使用的近似量

$$
M=n^2-\left\lfloor\frac{n+1}{2}\right\rfloor^2-n+1+
\left(p_0+\left\lfloor\frac n{p_0}\right\rfloor+
\left\lfloor\frac{n+1}{2}\right\rfloor\right)^2.
$$

对 $e/(M+1)$ 做连分数。若某个收敛分数为 $k/d$，则由 $ed-k\Phi=1$ 得到候选

$$
\Phi=\frac{ed-1}{k}.
$$

代回关于 $S$ 的二次方程求整数根，再解 $x^2-Sx+n=0$ 即可检验因子。完整常量直接取自题目输出，攻击部分如下：

```python
from sage.all import continued_fraction, gcd, solve, var


def recover_private_key(e, n, p0):
    half = (n + 1) // 2
    M = n**2 - half**2 - n + 1 + (p0 + n // p0 + half)**2
    cf = continued_fraction(e / (M + 1))

    for i in range(min(len(cf), 10000) + 1):
        k = cf.numerator(i)
        d = cf.denominator(i)
        if k == 1 or (e * d - 1) % k:
            continue

        phi = (e * d - 1) // k
        S = var("S")
        for sol in solve(S**2 + (n + 1) * S + n**2 - n + 1 == phi,
                         S, solution_dict=True):
            s = sol[S]
            if not s.is_integer():
                continue
            x = var("x")
            roots = solve(x**2 - s * x + n == 0, x, solution_dict=True)
            for root in roots:
                p = int(root[x])
                if p > 1 and n % p == 0:
                    return int(d), p, n // p
    raise ValueError("no valid convergent")


d, p, q = recover_private_key(e, n, p0)
assert p * q == n
m1, m2 = special_power(c, d, n)
left = long_to_bytes(m1).split(b"#", 1)[0]
right = long_to_bytes(m2).split(b"#", 1)[0]
print(left + right)
```

脚本恢复出：

```text
miniLCTF{CoNt1nu4d_FrActiOn_1s_3o_E@s7_foR_y0u!}
```

本题所用结论对应论文中的 Wiener 型小私钥指数攻击及其带近似因子的改进版本；正文已经写明攻击所需的群阶关系、近似量和验证步骤，因此不依赖外链才能复现。进一步的理论证明可参考 [Wiener-type attack](https://doi.org/10.1016/j.tcs.2021.06.033) 与 [改进的小私钥指数攻击](https://doi.org/10.1016/j.tcs.2022.05.010)。

## 方法总结

“not RSA”并不意味着 RSA 攻击思路失效：只要仍存在 $ed-k\Phi=1$ 且 $d$ 足够小，就应检查连分数近似。真正需要重建的是新群的阶函数，而不是机械套用 $(p-1)(q-1)$。得到候选后必须用整数根和 $pq=n$ 双重验证，最后再调用题目定义的 `special_power`，不能用普通模幂代替。
