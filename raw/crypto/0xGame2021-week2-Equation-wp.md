# week2Equation

## 题目简述

服务端每次生成新的 1408 bit RSA 素数 $p,q$，返回 $n,e,c$，以及随机小系数 $a,b$ 和线性关系

$$
f=ap+bq.
$$

目标是利用这条额外关系分解 $n=pq$，再完成 RSA 解密。

## 解题过程

将 $p=n/q$ 代入 $f=ap+bq$：

$$
bq^2-fq+an=0.
$$

该二次方程的判别式为

$$
\Delta=f^2-4abn=(ap-bq)^2,
$$

所以 $\Delta$ 必然是完全平方数。直接计算整数平方根，比在超大区间内做浮点求根或依赖单调区间二分更简洁、精确。两个形式根为

$$
q=\frac{f\pm\sqrt\Delta}{2b}.
$$

其中只有实际素因子会是整数并整除 $n$，逐个验证即可：

```python
from math import isqrt
from Crypto.Util.number import inverse, long_to_bytes

def solve(n, e, c, a, b, f):
    delta = f * f - 4 * a * b * n
    root = isqrt(delta)
    assert root * root == delta

    p = q = None
    for signed_root in (root, -root):
        numerator = f + signed_root
        denominator = 2 * b
        if numerator % denominator:
            continue
        candidate = numerator // denominator
        if candidate > 1 and n % candidate == 0:
            q = candidate
            p = n // q
            break

    if p is None:
        raise ValueError("未找到 n 的因子，请检查 a、b、f 的抄录顺序")

    d = inverse(e, (p - 1) * (q - 1))
    return long_to_bytes(pow(c, d, n))

# 将当前连接返回的六个整数填入后运行：
# print(solve(n, e, c, a, b, f).decode())
```

源码中的 flag 每次连接动态生成，格式为：

```text
0xGame{JieFang[三位数字]ChengHen[三位0-7数字]YouQu_GaoShu[三位十六进制]YeRuCi!}
```

因此仓库列出的多个 flag 都只是合法样例，不存在一个固定的唯一字符串。

## 方法总结

当 RSA 同时泄露 $n=pq$ 与 $ap+bq$ 时，应消元构造二次方程；其判别式天然等于平方数 $(ap-bq)^2$。使用 `isqrt` 并检查“完全平方、整除 $n$”两项条件，可避免浮点误差和错误根。
