# Do Math!

## 题目简述

题目使用标准 RSA 加密，却额外输出五个“提示”。生成方式是把 $p,q,e,n,d$ 分别乘以随机素数后对 $n$ 取模；其中与 $n$ 对应的提示本应为零，代码又把零改成了 $n-1$。这些提示泄露了模数因子。

## 解题过程

设第一个提示为

$$
h_p=(p\cdot r)\bmod n,
$$

且 $n=pq$。因为 $h_p$ 仍含有因子 $p$，通常有 $\gcd(h_p,n)=p$。题目没有直接另给 $n$，但第四个提示由零改成了 $n-1$，所以 $n=h_n+1$。同理可恢复 $q$：

```python
from math import gcd
from Crypto.Util.number import long_to_bytes

c = ...        # hints.txt 第一行的密文
hints = [...]  # “Hints:” 后的五个整数
n = hints[3] + 1
p = gcd(hints[0], n)
q = gcd(hints[1], n)
assert p * q == n

e = 65537
phi = (p - 1) * (q - 1)
d = pow(e, -1, phi)
m = pow(c, d, n)
print(long_to_bytes(m))
```

恢复的明文是：

```text
byuctf{th3_g00d_m4th_1snt_th4t_h4rd}
```

## 方法总结

“随机数乘秘密再模 $n$”并不会隐藏与 $n$ 的公因子。看到模运算提示时，应检查 `gcd(hint, n)`；同时要完整审计特殊值处理，本题正是 `0 -> n-1` 的分支反向泄露了 $n$。
