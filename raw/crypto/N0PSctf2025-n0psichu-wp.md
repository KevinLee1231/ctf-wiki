# n0psichu

## 题目简述

服务生成两个 512 位素数 $p,q$，公开模数 $N=pq$ 和加密后的 flag。`Polish the keys` 功能还会输出若干个与某个私钥素数近似成倍数关系的整数，需要利用这些泄漏分解 $N$，再按自定义密码体制的逆运算恢复明文。

## 解题过程

泄漏函数为：

```python
def polish(secret_key, count):
    bit_length = secret_key[0].bit_length()
    return [
        secret_key[getRandomRange(0, 2)] * getPrime(bit_length)
        + getRandomNBitInteger(bit_length >> 1)
        for _ in range(count)
    ]
```

因此每个样本都满足：

$$
a_i=pq_i+r_i
$$

或

$$
a_i=qq_i+r_i
$$

其中乘数约为 512 位，而误差 $r_i$ 只有 256 位。这正是近似公因数问题（Approximate Common Divisor，ACD）的结构。连接服务后，先通过 `Informations` 保存 $N$ 与 `encrypted_flag`，再让 `Polish` 尽可能返回 16 个样本。由于每个样本随机选择 $p$ 或 $q$，不能把所有样本直接视为拥有同一个近似公因数；官方脚本枚举 4 元组合，总能较快找到一组来自同一素数的样本。

对候选组合构造多元多项式：

$$
f_i(x_i)=x_i-a_i
$$

真实根就是小误差 $r_i$，并且 $f_i(r_i)$ 都是秘密素数的倍数。以误差界 $R=2^{256}$ 对变量缩放，生成包含 $N$ 倍数的多项式移位，经过 LLL 约化后重建在整数上成立的短多项式，再用 Gröbner 基求出各个 $r_i$。最后计算：

$$
\gcd(N,a_i-r_i)
$$

即可得到 $p$ 或 $q$。仓库中的 `sol.py` 已实现这一 ACD 流程，核心调用关系如下：

```python
from itertools import combinations

rho = 256

for samples in combinations(polished_values, 4):
    result = attack(
        modulus,
        samples,
        rho,
        t=1,
        k=1,
        roots_method="groebner",
    )
    if result is None or result[0] == 1:
        continue

    p = result[0]
    q = modulus // p
    assert p * q == modulus
    break
```

分解模数后，需要按源码的自定义运算解密，而不能套用标准 RSA 密文格式。密文为 `((c1, c2), f)`，其逆运算如下：

```python
from Crypto.Util.number import inverse, long_to_bytes


def decrypt(encrypted, secret_key):
    p, q = secret_key
    (c1, c2), f = encrypted
    modulus = p * q
    private_exponent = inverse(
        65537,
        (p - 1) * (q - 1),
    )

    a = pow(f, private_exponent, modulus)
    assert (
        c1**2 - a**2 * c2**2
    ) % modulus == 1

    c = pow(
        c1 - a * c2,
        private_exponent,
        modulus,
    )
    message = (
        (inverse(c, modulus) - c)
        * inverse(2 * a, modulus)
    ) % modulus
    return long_to_bytes(message)


print(decrypt(encrypted_flag, (p, q)))
```

最终得到：

```text
N0PS{RSA_P3lL_cRyp70_5YSTeM!}
```

## 方法总结

本题的决定性漏洞来自带小误差的私钥倍数泄漏。识别 ACD 结构后，还要注意样本混合了两个秘密素数，因此应先做小规模组合，再对每组运行格攻击。成功分解模数只是第一步；面对自定义加密方案，还必须从源码还原密文各分量的代数关系，使用与加密过程匹配的逆运算。
