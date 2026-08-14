# CakeCTF 2022 brand new crypto Writeup

## 题目简述

题目把 flag 按字节拆开。对每个明文字节 $m$ 独立选择随机数 $r$，再输出一对密文：

$$
c_1=m r^s \bmod n,\qquad c_2=m r^t \bmod n.
$$

公开参数是 $(s,t,n)$，私钥 $(a,b,n)$ 不给出。目标是在不知道 $r$、$a$、$b$ 以及 $n$ 的分解的情况下恢复每个明文字节。

关键弱点不在 RSA 分解，而在明文空间只有一个字节。两个密文分量还复用了同一个随机数 $r$，因此可以消掉随机因子并验证明文猜测。

## 解题过程

对一个候选明文字节 $m$，由第一项可得：

$$
c_1m^{-1}=r^s\pmod n.
$$

将它提升到 $t$ 次方：

$$
(c_1m^{-1})^t=r^{st}\pmod n.
$$

另一方面，第二项提升到 $s$ 次方后为：

$$
c_2^s=(mr^t)^s=m^sr^{st}\pmod n.
$$

两式相除即可消掉 $r^{st}$：

$$
c_2^s\left((c_1m^{-1})^t\right)^{-1}=m^s\pmod n.
$$

所以只需枚举可打印 ASCII 字节，检查等式是否成立。这里的候选字节均远小于两个 512 位素因子，因而都能在模 $n$ 下求逆。

```python
from ast import literal_eval

with open("output.txt", "r", encoding="utf-8") as f:
    pubkey = literal_eval(f.readline())
    ciphertexts = literal_eval(f.readline())

s, t, n = pubkey
flag = bytearray()

for c1, c2 in ciphertexts:
    for m in range(0x20, 0x7f):
        r_s = c1 * pow(m, -1, n) % n
        r_st = pow(r_s, t, n)
        m_s = pow(c2, s, n) * pow(r_st, -1, n) % n

        if pow(m, s, n) == m_s:
            flag.append(m)
            break
    else:
        raise ValueError("当前密文没有匹配的可打印字节")

print(flag.decode())
```

运行仓库中的官方 solver，可得到：

```text
CakeCTF{s0_anyway_tak3_car3_0f_0n3_byt3_p1aint3xt}
```

## 方法总结

这题的核心是“小明文空间 + 同一随机数生成两个相关密文”。即使整体加密形式看起来像带随机掩码的公钥方案，只要能构造一个仅依赖候选明文的等式，逐字节枚举就足够完成恢复。

分析此类方案时，应先把多个密文分量写成代数式，尝试通过乘方、相除或线性组合消去随机量；不要看到大模数就先入为主地尝试分解 $n$。
