# Intro to RSA 3

## 题目简述

题目使用 $N=pq$，但两个素数都满足低 600 位为 `...0001`。程序泄露 `sm = (N & (2^1100-1)) + 1`。目标是从这一低位信息恢复 $p+q$，进而用二次方程分解 $N$。

## 解题过程

写成 $p=a2^{600}+1$、$q=b2^{600}+1$，展开：

$$N=ab2^{1200}+(a+b)2^{600}+1$$

对 $2^{1100}$ 取低位时，第一项完全消失，而 $p+q=(a+b)2^{600}+2$。于是：

$$N\bmod2^{1100}+1=p+q$$

这正是题目给出的 `sm`。已知 $s=p+q$ 和 $N=pq$ 后，$p,q$ 是方程 $x^2-sx+N=0$ 的两根，判别式为：

$$\Delta=s^2-4N=(p-q)^2$$

因此可以直接恢复因子：

```python
from gmpy2 import iroot
from Crypto.Util.number import inverse, long_to_bytes

s = (N & ((1 << 1100) - 1)) + 1
delta = int(iroot(s * s - 4 * N, 2)[0])
p = (s + delta) // 2
q = N // p

phi = (p - 1) * (q - 1)
d = inverse(e, phi)
print(long_to_bytes(pow(c, d, N)))
```

结果为：

```text
grey{w0w_m4th_15_50_fun}
```

## 方法总结

固定素数的大段低位会让乘积的低位泄露对称量 $p+q$。RSA 一旦同时知道乘积与和，分解就退化为求解二次方程。分析自定义素数生成器时，应优先展开代数表达式并检查模 $2^k$ 的泄露。
