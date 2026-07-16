# week2RSA大闯关

## 题目简述

题目把同一份数据连续经过四层 RSA 变换，后一层的明文是前一层的密文。四层分别存在低指数小明文、特殊素数结构、共享素因子和共模攻击四种缺陷。附件只公开各层解题所需的参数，因此必须从第四层开始逆序还原，才能最终得到 flag。

## 解题过程

### 第四层：共模攻击

同一明文 $m$ 在相同模数 $n$ 下分别使用 $e_1=823$、$e_2=827$ 加密：

$$
c_1\equiv m^{e_1}\pmod n,\qquad c_2\equiv m^{e_2}\pmod n.
$$

因为 $\gcd(e_1,e_2)=1$，扩展欧几里得算法可求出 $u,v$，使 $ue_1+ve_2=1$，于是

$$
m\equiv c_1^u c_2^v\pmod n.
$$

若指数为负，要先对对应密文求模逆，不能直接按普通正指数处理。

### 第三层：共享素因子

源码分别构造了 $n_1=p_1q$ 和 $n_2=p_2q$。两个模数复用了素因子 $q$，所以

$$
q=\gcd(n_1,n_2).
$$

分解 $n_1$ 后即可计算私钥，解出第三层密文。公开的 `c_hint` 不是主链必需参数；真正的漏洞是两个模数不互素。

### 第二层：特殊模数结构

本层直接给出 $p$，源码还规定 $q=2p-1$，且

$$
n=p^2q^3r.
$$

因此可整除得到 $r$，并按素数幂的欧拉函数计算

$$
\varphi(n)=p(p-1)q^2(q-1)(r-1).
$$

### 第一层：低指数小明文

第一层使用 $e=5$，但没有填充。已恢复的密文满足 $c\equiv m^5\pmod n$，因此存在非负整数 $k$ 使

$$
m^5=c+kn.
$$

从 $k=0$ 开始寻找使右侧成为完全五次幂的值即可。

下面的脚本直接解析附件 `output.txt`，无需在 WP 中重复粘贴大整数：

```python
from itertools import count
from math import gcd

from Crypto.Util.number import long_to_bytes
from gmpy2 import iroot


def exgcd(a, b):
    if b == 0:
        return a, 1, 0
    g, x1, y1 = exgcd(b, a % b)
    return g, y1, x1 - (a // b) * y1


def signed_pow(base, exponent, modulus):
    if exponent >= 0:
        return pow(base, exponent, modulus)
    return pow(pow(base, -1, modulus), -exponent, modulus)


parts = {}
section = None
with open("output.txt", "r", encoding="utf-8") as f:
    for raw_line in f:
        line = raw_line.strip()
        if not line:
            continue
        if line.endswith(":"):
            section = line[:-1]
            parts[section] = {}
            continue
        key, value = line.split("=", 1)
        parts[section][key.strip()] = int(value.strip())

# part 4：共模攻击
e1, e2 = 823, 827
g, u, v = exgcd(e1, e2)
assert g == 1
n4 = parts["part4"]["n"]
cipher = (
    signed_pow(parts["part4"]["c1"], u, n4)
    * signed_pow(parts["part4"]["c2"], v, n4)
) % n4

# part 3：共享素因子
n1 = parts["part3"]["n1"]
n2 = parts["part3"]["n2"]
q = gcd(n1, n2)
p = n1 // q
phi = (p - 1) * (q - 1)
d = pow(0x10001, -1, phi)
cipher = pow(cipher, d, n1)

# part 2：n = p^2 * q^3 * r
p = parts["part2"]["p"]
n2_special = parts["part2"]["n"]
q = 2 * p - 1
r = n2_special // (p**2 * q**3)
assert n2_special == p**2 * q**3 * r
phi = p * (p - 1) * q**2 * (q - 1) * (r - 1)
d = pow(0x10001, -1, phi)
cipher = pow(cipher, d, n2_special)

# part 1：寻找 c + k*n 的精确五次根
n1_low_e = parts["part1"]["n"]
for k in count():
    root, exact = iroot(cipher + k * n1_low_e, 5)
    if exact:
        flag = long_to_bytes(int(root))
        print(flag.decode())
        break
```

运行结果：

```text
0xGame{rsa-----just_so_so}
```

## 方法总结

本题不是四道互相独立的 RSA 小题，而是一条串行加密链，所以解密顺序必须与加密顺序相反。识别点依次是：同模数同明文且指数互素、两个模数的最大公因数不为 1、模数含重复素数幂、低指数且无填充。最容易出错的是把 $p^2q^3r$ 的欧拉函数仍写成 $(p-1)(q-1)(r-1)$，以及在共模攻击中忽略负 Bézout 系数对应的模逆运算。
