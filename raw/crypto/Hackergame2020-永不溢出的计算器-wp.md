# Hackergame2020 永不溢出的计算器 WP

## 题目简述

计算器在未知模数 $n=pq$ 下提供加减乘除、乘方与模平方根，并给出 $c=m^{65537}\bmod n$。普通运算只需知道 $n$，真正泄露分解信息的是平方根 oracle。

## 解题过程

先查询：

```text
0 - 1
```

结果是 $n-1$，因此加一得到模数 $n$。随机选择 $x\in[2,n-2]$，计算：

$$
a=x^2\bmod n,
$$

再把 `sqrt(a)` 交给服务器，得到一个平方根 $y$。当 $n=pq$ 且 $p,q$ 为不同奇素数时，一个非零二次剩余通常有四个根：$\pm x$ 以及另一对根。若 $y\equiv x$ 或 $y\equiv -x\pmod n$，本轮没有新信息，重新随机取值即可。

若 $y\not\equiv\pm x\pmod n$，则：

$$
x^2\equiv y^2\pmod n
\quad\Longrightarrow\quad
n\mid(x-y)(x+y).
$$

两个因子都不是 $n$ 的倍数，故：

$$
p=\gcd(n,x-y)
$$

以约 $1/2$ 的概率给出 $n$ 的非平凡因子，随后令 $q=n/p$。解密步骤为：

```python
from math import gcd
from secrets import randbelow

while True:
    x = 2 + randbelow(n - 3)
    a = pow(x, 2, n)
    y = sqrt_oracle(a)
    p = gcd(n, x - y)
    if 1 < p < n:
        break

q = n // p
phi = (p - 1) * (q - 1)
d = pow(65537, -1, phi)
m = pow(c, d, n)
flag = m.to_bytes((m.bit_length() + 7) // 8, "big")
```

服务端源码用 `sqrt_mod` 枚举 $p$、$q$ 上的根并经 CRT 合并，然后返回最小根；“返回最小值”不会消除不同平方根带来的因式分解能力。

## 方法总结

RSA 模数上的通用平方根 oracle 等价于泄露分解能力。只要能为随机平方 $x^2$ 获得一个不等于 $\pm x$ 的根，就能通过最大公约数分解 $n$。密码接口不应暴露未经限制的代数逆运算，即使其他加减乘除看起来无害。
