# accountleak

## 题目简述

题目把密码作为 RSA 明文，给出模数 $n=pq$、密文 $c=m^e\bmod n$ 和额外泄漏

$$
L=(p-s)(q-s),
$$

其中 $e=65537$，而随机偏移 $s$ 只有 20 bit。服务要求在 2.5 秒内提交解出的密码，因此关键不是通用分解 512 bit 素数，而是利用小范围偏移把额外等式还原成因子。

## 解题过程

展开泄漏式并代入 $pq=n$：

$$
L=n-s(p+q)+s^2.
$$

对每个候选 $s\in[1,2^{20})$，令 $p=n/q$，整理得到关于 $q$ 的二次方程：

$$
s q^2+(L-n-s^2)q+sn=0.
$$

其判别式必须为完全平方数。找到整数根后，再用原泄漏式验证即可排除伪解。下面的核心脚本直接恢复私钥并解密：

```python
from math import isqrt
from Crypto.Util.number import inverse, long_to_bytes

def factor_from_leak(n: int, leak: int):
    for s in range(1, 1 << 20):
        # A*q^2 + B*q + C = 0
        A = s
        B = leak - n - s * s
        C = s * n
        delta = B * B - 4 * A * C
        if delta < 0:
            continue
        root = isqrt(delta)
        if root * root != delta:
            continue
        for numerator in (-B + root, -B - root):
            denominator = 2 * A
            if numerator % denominator:
                continue
            q = numerator // denominator
            if q > 1 and n % q == 0:
                p = n // q
                if (p - s) * (q - s) == leak:
                    return p, q
    raise ValueError("offset not found")

def decrypt(n: int, c: int, leak: int) -> bytes:
    p, q = factor_from_leak(n, leak)
    phi = (p - 1) * (q - 1)
    d = inverse(65537, phi)
    return long_to_bytes(pow(c, d, n))
```

连接服务后解析三个整数，把 `decrypt` 的返回值原样提交，服务返回：

```text
tjctf{h3y_wh3r3_d1d_my_d1am0nds_g0_th3y_w3r3_ju5t_h3r3}
```

## 方法总结

- 核心弱点是共享小偏移产生的代数泄漏，而不是 RSA 参数长度不足。
- 将 $pq=n$ 代入展开式后，可把未知因子化成一元二次方程；完全平方判别式提供了很强的筛选条件。
- 远程限时题应把连接和解析开销压到最低，并在提交前用原等式验证因子，避免偶然整数根造成错误答案。
