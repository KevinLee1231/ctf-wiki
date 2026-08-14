# Intro to RSA 2

## 题目简述

题目使用异常模数 $N=p^2q^3$。两个 1024 位素数不是均匀生成，而都满足 $p\equiv q\equiv1\pmod{2^{1000}}$，也就是仅高约 24 位未知。目标是利用这一结构分解 $N$，再按重复素因子的欧拉函数解密。

## 解题过程

生成函数把随机数 $x$ 左移 1000 位后加一：

```python
p = (x << 1000) + 1
```

因此候选素数形如 $k\cdot2^{1000}+1$，而 $k$ 的搜索空间只有约 $2^{24}$。逐个候选测试 `N % candidate == 0`，找到其中一个素因子后，再根据它在 $N$ 中的指数判断它是 $p$ 还是 $q$：

```python
from gmpy2 import iroot

candidate = 1
for _ in range(1, 1 << 24):
    candidate += 1 << 1000
    if N % candidate == 0:
        factor = candidate
        break

if N % (factor ** 3) == 0:
    q = factor
    p = int(iroot(N // q ** 3, 2)[0])
else:
    p = factor
    q = int(iroot(N // p ** 2, 3)[0])
```

对 $N=p^2q^3$，欧拉函数不是普通的 $(p-1)(q-1)$，而是：

$$\varphi(N)=p(p-1)q^2(q-1)$$

随后计算 $d=e^{-1}\bmod\varphi(N)$、$m=c^d\bmod N$ 并转为字节，得到：

```text
grey{c0ngr4ts_y0u_kn0w_brut3_f0rc3}
```

## 方法总结

素数位数大不等于搜索空间大。若大量低位被固定，真正熵值只剩高位的少量比特，就可以枚举。处理含重复素因子的 RSA 模数时，还必须按 $\varphi(p^a)=p^{a-1}(p-1)$ 计算欧拉函数。
