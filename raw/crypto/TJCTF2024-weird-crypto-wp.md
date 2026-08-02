# weird-crypto

## 题目简述

题目生成普通 RSA 模数 $n=pq$，但把一个 20 bit 素数命名为 `oops` 并作为公钥指数 $e$。随后计算私钥指数

$$
d=e^{-1}\bmod\varphi(n),
$$

却把 $d$ 和 $m^d\bmod n$ 一起输出。也就是说，附件中的 `hehe_secret` 实际上是用私钥指数做的 RSA 运算，而真正需要补回的公钥指数只有 20 bit。

## 解题过程

RSA 指数互逆关系给出

$$
(m^d)^e=m^{de}\equiv m\pmod n
$$

（对本题的合法明文成立）。因此不需要分解 $n$，也不需要利用公开的 $d$ 反推 $\varphi(n)$；只要枚举所有 20 bit 素数 $e$，计算 `pow(hehe_secret, e, n)`，再检查结果是否包含 flag 格式。

```python
from Crypto.Util.number import isPrime, long_to_bytes

def recover(n: int, transformed: int):
    for e in range(1 << 19, 1 << 20):
        if not isPrime(e):
            continue
        candidate = long_to_bytes(pow(transformed, e, n))
        if candidate.startswith(b"tjctf{") and candidate.endswith(b"}"):
            return e, candidate
    raise ValueError("public exponent not found")
```

枚举后得到：

```text
tjctf{congrats_on_rsa_e_djfkel2349!}
```

## 方法总结

- 变量名和“加密/解密”标签不重要，RSA 中真正决定逆运算的是指数关系 $ed\equiv1\pmod{\varphi(n)}$。
- 当未知公钥指数只有 20 bit 时，直接枚举素数比尝试分解模数更自然、更快速。
- 已知 flag 前后缀提供了确定的离线判据；找到候选后仍应重新做一次指数运算，确认明文与题目输出完全一致。
