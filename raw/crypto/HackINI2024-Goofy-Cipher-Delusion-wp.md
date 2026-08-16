# Goofy Cipher Delusion

## 题目简述

题目使用 RSA 模数 $N=PQ$ 和指数 `e = 131`，同时公开正常密文 $c=m^e\bmod N$，以及在素因子 `P` 下重复加密得到的 $c_p=m^e\bmod P$。后者看似只是“再加密一次”，实际上直接泄露了 `P` 的整除关系。

## 解题过程

由于 $P\mid N$，正常 RSA 密文对 `P` 取模后满足：

$$
c\equiv m^e\pmod P
$$

而附件又给出：

$$
c_p\equiv m^e\pmod P
$$

两式相减可知 $P\mid(c-c_p)$，所以：

$$
P=\gcd(N,c-c_p)
$$

得到因子后按普通 RSA 解密即可：

```python
from Crypto.Util.number import long_to_bytes
from pathlib import Path
from math import gcd

values = {}
for line in Path("out.txt").read_text().splitlines():
    name, value = line.split("=", 1)
    values[name.strip()] = int(value)

N = values["N"]
e = values["e"]
c = values["c"]
cp = values["cp"]

P = gcd(N, c - cp)
assert 1 < P < N and N % P == 0
Q = N // P
phi = (P - 1) * (Q - 1)
d = pow(e, -1, phi)
m = pow(c, d, N)
print(long_to_bytes(m))
```

对 `challenge/out.txt` 中的数据运行后得到：

```text
shellmates{t00_3asy_4_a_gcd_m4st3r}
```

## 方法总结

- 核心技巧：将同一个幂在复合模数和其素因子模数下的结果作差，再与 `N` 求最大公因数。
- 识别信号：RSA 题额外泄露 $m^e\bmod P$、$m^e\bmod Q$ 或与其同余的值时，通常可直接构造因子整除关系。
- 复用要点：求得的 GCD 应验证既非 1 也非 `N`；随后还要确认 `e` 在 $\varphi(N)$ 下可逆。
