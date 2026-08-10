# RSA 大冒险 1

## 题目简述

题目由四个连续的 RSA 小关组成，每关都要求提交一个字符串。四关分别考查：已知一个因子且明文小于该因子、重复使用素因子、低加密指数，以及共模攻击。全部通过后，服务返回最终 flag。

## 解题过程

### 第一关：利用已知素因子

模数满足 $n=pqr$，题目同时给出 $p$、$e$ 和密文 $c$，并保证明文整数 $m<p$。将 RSA 等式缩小到模 $p$ 的范围：

$$
c\equiv m^e\pmod p.
$$

计算 $d_p=e^{-1}\bmod(p-1)$ 后，有

$$
m\equiv c^{d_p}\pmod p.
$$

由于 $m<p$，这个模 $p$ 的结果就是原始明文。该关提交：

```text
m<n_But_also_m<p
```

### 第二关：重复使用素因子

服务每次加密都会生成新的 $q$，但错误地重复使用同一个 $p$。请求两组数据 $(n_1,c_1)$ 和 $(n_2,c_2)$，即可计算：

$$
p=\gcd(n_1,n_2).
$$

随后用 $q_1=n_1/p$ 分解第一组模数，计算私钥并解密。该关提交：

```text
make_all_modulus_independent
```

### 第三关：低加密指数

这一关使用 $e=3$ 且没有填充。若 $m^3<n$，则模运算没有发生回绕，密文就是普通整数立方：

$$
c=m^3.
$$

直接取整数立方根即可恢复明文；若边界条件下发生少量回绕，可依次检查 $c+kn$ 是否为完全立方数。该关提交：

```text
encrypt_exponent_should_be_bigger
```

### 第四关：共模攻击

同一明文在相同模数 $n$ 下分别使用互素指数 $e_1,e_2$ 加密，得到 $c_1,c_2$。由扩展欧几里得算法求出 $s,t$，使

$$
se_1+te_2=1.
$$

于是：

$$
m\equiv c_1^s c_2^t\pmod n.
$$

指数为负数时，应先求对应密文的模逆元。下面的框架覆盖四关的核心计算：

```python
from math import gcd


def long_to_bytes(value: int) -> bytes:
    size = max(1, (value.bit_length() + 7) // 8)
    return value.to_bytes(size, "big")


def egcd(a: int, b: int) -> tuple[int, int, int]:
    if b == 0:
        return a, 1, 0
    g, x, y = egcd(b, a % b)
    return g, y, x - (a // b) * y


def signed_pow(base: int, exponent: int, modulus: int) -> int:
    if exponent < 0:
        base = pow(base, -1, modulus)
        exponent = -exponent
    return pow(base, exponent, modulus)


def decrypt_with_known_p(c: int, e: int, p: int) -> bytes:
    dp = pow(e, -1, p - 1)
    return long_to_bytes(pow(c, dp, p))


def decrypt_reused_prime(n1: int, c1: int, n2: int, e: int) -> bytes:
    p = gcd(n1, n2)
    q = n1 // p
    d = pow(e, -1, (p - 1) * (q - 1))
    return long_to_bytes(pow(c1, d, n1))


def common_modulus(n: int, e1: int, c1: int, e2: int, c2: int) -> bytes:
    g, s, t = egcd(e1, e2)
    assert g == 1
    m = signed_pow(c1, s, n) * signed_pow(c2, t, n) % n
    return long_to_bytes(m)
```

第四关按题目原文提交拼写为：

```text
never_uese_same_modulus
```

完成四关后得到：

```text
hgame{W0w_you^knowT^e_CoMm0n_&t$ack_@bout|RSA}
```

官方总 WP 没有保留交互实例的全部大整数。最终 flag 与四关提交字符串可用参赛者复盘文章交叉核对：[Mar10 的 HGAME 2023 复盘](https://www.cnblogs.com/Mar10/p/17063246.html) 与 [App1eTree 的 HGAME 2023 题解](https://www.cnblogs.com/App1eTree/p/2023hgame.html)。上述攻击原理和计算过程已完整写入正文，解题不依赖外链。

## 方法总结

四关分别对应 RSA 中常见的结构性错误：泄露可用模因子、跨密钥复用素数、不填充的小指数加密，以及相同模数复用。检查 RSA 题时，应优先审计参数之间是否共享因子、指数是否过小、明文是否小于已知因子，以及同一明文是否在共模下被多次加密。
